# 🌿 Carbon Markets Advisor — Build Blueprint

> A step-by-step engineering plan to build a local-first AI document intelligence system for forestry carbon projects. Read this before writing a single line of code.

---

## What We're Building

Users upload messy, varied forestry documents (PDFs, Word docs, scanned reports — anything). The system:

1. Reads and understands the documents
2. Extracts carbon-project-specific knowledge (dynamically — no fixed schema)
3. Lets users chat with their documents like a senior advisor
4. Generates structured advisory reports

This is **not a chatbot wrapper**. It's a domain-specific RAG (Retrieval-Augmented Generation) system with a carbon markets knowledge layer baked in.

---

## Multi-Model Strategy (OpenRouter)

Different tasks need different models. Free tier on OpenRouter means we route carefully.

| Task | Model | Why |
|---|---|---|
| Document extraction (structured JSON) | `qwen/qwen-2.5-72b-instruct:free` | Strong instruction-following, good JSON output |
| Advisory chat (nuanced answers) | `meta-llama/llama-3.1-70b-instruct:free` | Natural language quality |
| Report generation (long-form) | `mistralai/mistral-7b-instruct:free` | Fast, decent for structured prose |
| Fallback (when others rate-limit) | `google/gemma-2-9b-it:free` | Lightweight backup |

> **Rule:** Never hardcode a model name in business logic. Always route through a central `ModelRouter` class with fallback logic.

---

## Phased Build Plan

---

### Phase 0 — Foundation (Day 1)
**Goal:** Postgres + pgvector running, env wired up, FastAPI server starts.

#### Tasks
- [ ] Set up PostgreSQL locally with pgvector extension
- [ ] Create `.env` from `.env.example`
- [ ] Install dependencies
- [ ] Write `app/core/config.py` (Settings class)
- [ ] Write `app/db/database.py` (engine + session)
- [ ] Write `app/main.py` (bare FastAPI, `/health` endpoint only)
- [ ] Confirm: `GET /health` returns `{"status": "ok"}`

#### Acceptance Criteria
```bash
# This must work before moving to Phase 1
curl http://localhost:8000/health
# → {"status": "ok", "env": "development"}

# pgvector must be available
psql -d carbon_advisor -c "CREATE EXTENSION IF NOT EXISTS vector;"
# → CREATE EXTENSION
```

#### Edge Cases
- `DATABASE_URL` has wrong password → config loads, but DB calls fail at runtime. Catch this in lifespan startup with a test query.
- pgvector not installed → clear error at startup, not a cryptic crash.
- Missing env vars → pydantic-settings raises `ValidationError` immediately, not silently.

---

### Phase 1 — Document Ingestion (Day 2–3)
**Goal:** Upload a PDF/DOCX/TXT, extract its text, store chunks in DB.

#### Tasks
- [ ] Write `app/utils/file_parser.py` — PDF, DOCX, TXT extraction
- [ ] Write `app/services/document_processor.py` — token-bounded chunking
- [ ] Write ORM models: `Document`, `DocumentChunk`
- [ ] Write `POST /documents/upload` route
- [ ] Write `GET /documents/{id}` status route
- [ ] Processing runs in a **background task** (don't block HTTP response)

#### How Chunking Works

```
Raw text (50,000 chars)
       ↓
Sentence split
       ↓
Group sentences until ~400 tokens
       ↓
Overlap: carry last sentence into next chunk (context continuity)
       ↓
Result: ~120 chunks of 300–400 tokens each
```

**Why 400 tokens?** With `max_context_chunks=6`, that's 2,400 tokens of context per LLM call — fits comfortably in any free model's context window.

#### Acceptance Criteria
```bash
# Upload a PDF
curl -X POST /documents/upload \
  -F "project_id=abc" \
  -F "file=@test.pdf"
# → {"document_id": "...", "status": "pending"}

# Poll status
curl /documents/{id}
# → {"status": "ready", "page_count": 24, "char_count": 48200}

# Verify chunks in DB
psql -c "SELECT COUNT(*) FROM document_chunks WHERE document_id='...';"
# → should be 50–200 chunks for a 20-page doc
```

#### Edge Cases
| Scenario | Expected Behavior |
|---|---|
| Empty PDF (scanned image only) | `status: error`, message: "No text extracted. Try OCR first." |
| `.exe` file uploaded | 400 Bad Request before any processing |
| 200MB PDF | 413 too large before reading bytes |
| PDF with 1,000 pages | Chunked fine; sampled to 40 chunks for analysis |
| DOCX with only a table | Tables often become empty paragraphs in python-docx. Detect this and warn. |
| Duplicate upload of same file | Still creates a new Document record (user may upload revision) |
| Background task crashes | Document status set to `"error"` with `error_message` populated |

---

### Phase 2 — Embeddings + Vector Search (Day 3–4)
**Goal:** Every chunk has a 1024-dim Cohere embedding stored in pgvector. Vector similarity search works.

#### Tasks
- [ ] Write `app/services/embedding_service.py`
- [ ] Embed chunks during upload pipeline (after chunking)
- [ ] Write `app/services/retrieval_service.py` — cosine search + Cohere rerank
- [ ] Add pgvector index for performance: `HNSW` or `IVFFlat`

#### Embedding Flow

```
chunks = ["Namizimu forest covers 45,000 ha...", "Land tenure is community-held..."]
         ↓
Cohere embed-english-v3.0 (input_type="search_document")
         ↓
vectors = [[0.023, -0.14, ...], [0.087, 0.003, ...]]  ← 1024 dims each
         ↓
Stored in document_chunks.embedding (pgvector Vector column)
```

**For queries:**
```
user_query = "What methodology fits this project?"
         ↓
Cohere embed (input_type="search_query")  ← DIFFERENT input_type!
         ↓
pgvector cosine distance search
         ↓
Top 12 candidates → Cohere rerank → Top 6 returned
```

> `input_type` matters. Cohere trains separate representations for documents vs queries. Using the wrong one degrades retrieval quality by ~15–20%.

#### Acceptance Criteria
```bash
# After uploading a doc, all chunks must have embeddings
psql -c "SELECT COUNT(*) FROM document_chunks WHERE embedding IS NULL;"
# → 0 (all embedded)

# Retrieval returns sensible results
# (test via /chat endpoint in Phase 4)
```

#### Edge Cases
| Scenario | Expected Behavior |
|---|---|
| Cohere API rate limit (free tier: 100 calls/min) | Exponential backoff with tenacity, max 3 retries |
| Empty chunk text (whitespace only) | Skip embedding; don't send to Cohere |
| Cohere returns fewer embeddings than texts | Raise immediately — never silently misalign vectors with chunks |
| pgvector column is NULL for some chunks | Retrieval query filters `WHERE embedding IS NOT NULL` |
| Very long single sentence (>400 tokens) | Hard word-split fallback in chunker |

#### pgvector Index (run once after loading data)
```sql
-- For datasets under ~100k chunks, IVFFlat is fine
CREATE INDEX ON document_chunks 
  USING ivfflat (embedding vector_cosine_ops)
  WITH (lists = 100);
```

---

### Phase 3 — Carbon Agent: Document Analysis (Day 4–5)
**Goal:** After upload, the agent automatically extracts structured project knowledge using a two-pass LLM pipeline.

#### The Two-Pass Architecture

**Why two passes?** Sending a 100-page document to an LLM at once costs 50,000+ tokens. Two passes costs ~8,000 total.

```
Pass 1: Per-chunk extraction (parallel-able)
─────────────────────────────────────────────
chunk_1 (400 tokens) → LLM → JSON extraction (200 tokens out)
chunk_2 (400 tokens) → LLM → JSON extraction (200 tokens out)
...  (sample max 40 chunks from the document)

Pass 2: Synthesis (single call)
─────────────────────────────────────────────
[extraction_1, extraction_2, ... extraction_40] → LLM → merged project profile
```

#### What Gets Extracted (Dynamic Schema)

The LLM is NOT given a fixed list of fields. It's told to extract what it finds. Base fields include:

```json
{
  "project_type": "REDD+",
  "location": "Namizimu, South Kivu, DRC",
  "country": "Democratic Republic of Congo",
  "forest_type": "tropical moist forest",
  "land_tenure": "community-held, contested",
  "carbon_rights": "not documented",
  "deforestation_drivers": ["charcoal production", "subsistence agriculture"],
  "proposed_activities": ["reduced deforestation", "community patrols"],
  "baseline_scenario": "continued deforestation at 2.3% per year",
  "carbon_standard_mentioned": "Verra VCS",
  "estimated_area_ha": 45000,
  "risks": ["land tenure unresolved", "no baseline study"],
  "missing_data": ["carbon stock inventory", "FPIC documentation"],
  "important_entities": ["MECNT", "WWF DRC"],
  "additional_fields": {
    "concession_number": "CN-2021-0847",
    "biodiversity_value": "high — presence of okapi"
  }
}
```

`additional_fields` is where the LLM puts anything it discovers that isn't in the base schema. This is the "dynamic" part.

#### Hallucination Guardrails

The extraction prompt explicitly requires:
- `(extracted)` — value comes directly from the document
- `(inferred)` — reasonable inference from context
- `null` — not present in the document

Example in output:
```json
{
  "land_tenure": "community-held (inferred)",
  "concession_number": null
}
```

#### Acceptance Criteria
```bash
# After uploading a project PDF
curl /projects/{id}
# Response must include a non-empty profile_json
# Must include "missing_data" array
# Must NOT contain invented place names or numbers not in the document
```

#### Edge Cases
| Scenario | Expected Behavior |
|---|---|
| LLM returns malformed JSON | Strip markdown fences, retry once, else store `{}` with error flag |
| All 40 chunk extractions return null for location | Synthesis reflects this — location stays null, not invented |
| Document is in French (common in DRC) | Qwen handles French. Note in profile: `"document_language": "fr"` |
| Document mentions multiple project areas | Synthesis lists all; flags ambiguity in `risks` |
| Re-analysis triggered (new docs added) | Profile is upserted (updated, not duplicated) |

---

### Phase 4 — Chat (RAG Pipeline) (Day 5–6)
**Goal:** Users can ask questions and get grounded, cited answers.

#### Full Chat Pipeline

```
User: "What methodology fits this project?"
           ↓
1. Embed query (Cohere, search_query)
           ↓
2. pgvector search → top 12 project chunks
           ↓
3. Cohere rerank → top 6 chunks
           ↓
4. Retrieve knowledge base chunks (top 3, from methodology docs)
           ↓
5. Load project profile JSON
           ↓
6. Build prompt:
   System: "You are a carbon markets advisor..."
   User:   [project profile] + [6 evidence chunks] + [3 knowledge chunks] + [question]
           ↓
7. LLM (Llama 3.1 70B via OpenRouter) → answer
           ↓
8. Return: {"answer": "...", "sources": [...]}
```

#### Response Format Contract

The LLM is instructed to prefix claims:
- `✅` — fact from document
- `⚠️` — missing information
- `🔮` — assumption/inference
- `📋` — methodology requirement

Example answer:
```
✅ The project area is 45,000 ha of tropical moist forest in South Kivu (Project Report p.3).

📋 Verra VCS REDD+ methodology (VM0007) requires a deforestation baseline using at 
least 10 years of historical satellite data.

⚠️ No baseline study was found in the uploaded documents. This is a critical gap.

🔮 Given the forest type and deforestation drivers mentioned, VM0007 or VM0015 are 
likely the most applicable methodologies, but this cannot be confirmed without a 
full methodology eligibility assessment.
```

#### Acceptance Criteria
```bash
curl -X POST /chat \
  -d '{"project_id": "...", "message": "What are the main risks?"}'

# Response must:
# 1. Cite at least one source document
# 2. Use the ✅/⚠️/🔮/📋 notation
# 3. Not claim facts not present in sources
# 4. Include non-empty "sources" array
```

#### Edge Cases
| Scenario | Expected Behavior |
|---|---|
| No documents uploaded yet | 400: "Upload and process documents first" |
| Question is off-topic ("write me a poem") | Answer politely redirects: "I can only advise on carbon project topics" |
| Question references a document page that wasn't sampled | Honest: "I don't have enough information from the documents to answer this" |
| Knowledge base is empty (no methodology docs loaded) | Still works, but `knowledge_chunks` section in prompt is empty |
| Very long question (>500 chars) | Truncate to 500 chars; log warning |
| LLM rate-limited | ModelRouter tries fallback model |

---

### Phase 5 — Model Router (Day 6)
**Goal:** A central class that routes to the right model per task, handles rate limits, and falls back gracefully.

#### ModelRouter Design

```python
class ModelRouter:
    TASK_MODELS = {
        "extraction":   ["qwen/qwen-2.5-72b-instruct:free", "google/gemma-2-9b-it:free"],
        "chat":         ["meta-llama/llama-3.1-70b-instruct:free", "qwen/qwen-2.5-72b-instruct:free"],
        "report":       ["mistralai/mistral-7b-instruct:free", "meta-llama/llama-3.1-70b-instruct:free"],
        "synthesis":    ["qwen/qwen-2.5-72b-instruct:free", "mistralai/mistral-7b-instruct:free"],
    }

    async def complete(self, task: str, messages: list, **kwargs) -> str:
        models = self.TASK_MODELS.get(task, self.TASK_MODELS["chat"])
        for model in models:
            try:
                return await self._call(model, messages, **kwargs)
            except RateLimitError:
                logger.warning(f"Rate limited on {model}, trying next...")
                continue
        raise AllModelsExhaustedError("All models rate-limited. Try again in a minute.")
```

#### Why This Matters
- Free OpenRouter models have per-minute token limits (~20k tokens/min per model)
- Different models have different strengths (Qwen for JSON, Llama for prose)
- If one model is down, the system still works

#### Edge Cases
| Scenario | Expected Behavior |
|---|---|
| All models rate-limited | Return 429 with `retry_after_seconds` hint |
| Model returns empty response | Treat as failure, try next model |
| Model returns response in wrong language | Log, return as-is (don't re-call — too expensive) |
| OpenRouter API key expired | 401 caught early, return 500 with clear message |

---

### Phase 6 — Knowledge Base (Day 7)
**Goal:** Methodology reference documents (Verra, ART TREES, etc.) are embedded and searchable, providing the "expert layer" on top of user documents.

#### Knowledge Base Structure

```
knowledge/
├── verra/
│   ├── VM0007_REDD_methodology.pdf      ← Verra REDD+ methodology
│   └── VCS_Standard_v4.pdf
├── art_trees/
│   └── ART_TREES_v2.0.pdf
├── gold_standard/
│   └── GS_Land_Use_Activity.pdf
├── article6/
│   └── Paris_Agreement_Article6_guidance.pdf
└── drc_redd/
    └── DRC_REDD_National_Strategy.pdf
```

> You don't need all of these on day one. Start with a few key Verra methodology summaries. Even a 5-page summary of VM0007 dramatically improves chat quality.

#### Loading Script

```bash
python -m scripts.load_knowledge
# Reads all files from knowledge/
# Chunks, embeds, stores in knowledge_chunks table
# Skips already-loaded files (idempotent)
```

#### Acceptance Criteria
```bash
# After loading, knowledge chunks exist
psql -c "SELECT source, COUNT(*) FROM knowledge_chunks GROUP BY source;"
#  source     | count
# ------------|-------
#  verra      |   847
#  art_trees  |   312
```

---

### Phase 7 — Report Generation (Day 8)
**Goal:** Generate a structured advisory report as markdown, grounded in project evidence.

#### Report Structure
```
## Executive Summary
## Project Classification
## Methodology Fit Assessment  
## Recommended Pathway
## Key Risks and Red Flags
## Missing Data Checklist
## Next Steps
```

#### Token Budget for Report
- Project profile: 1,500 tokens
- Evidence chunks (6 × 400): 2,400 tokens
- Knowledge chunks (4 × 400): 1,600 tokens
- System prompt: 300 tokens
- **Total input: ~5,800 tokens**
- Max output: 2,000 tokens
- **Total per report call: ~7,800 tokens**

This fits in free-tier context windows and costs very little.

---

## Full Data Model

```
User (id, email, hashed_password)
 └── Project (id, user_id, name, description)
      ├── ProjectProfile (project_id, profile_json, last_analyzed_at)
      └── Document (id, project_id, filename, file_type, status, page_count)
           └── DocumentChunk (id, document_id, project_id, chunk_index, text, embedding)

KnowledgeDocument (id, source, title, filename)
 └── KnowledgeChunk (id, document_id, source, chunk_index, text, embedding)
```

**Why `project_id` is denormalised onto `DocumentChunk`?**
Allows `WHERE project_id = ?` without a JOIN on every vector search — important for performance.

---

## API Contract (Final)

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/projects` | Create a project |
| `GET` | `/projects/{id}` | Get project + extracted profile |
| `POST` | `/projects/{id}/analyze` | Re-run analysis |
| `POST` | `/projects/{id}/report` | Generate advisory report |
| `POST` | `/documents/upload` | Upload document (starts pipeline) |
| `GET` | `/documents/{id}` | Check processing status |
| `POST` | `/chat` | Ask a question about a project |
| `GET` | `/health` | Health check |

---

## Token Budget Summary

| Operation | Input Tokens | Output Tokens | Total |
|---|---|---|---|
| Chunk extraction (per chunk) | ~500 | ~200 | ~700 |
| Per-doc analysis (40 chunks sampled) | ~8,000 | ~2,000 | ~10,000 |
| Synthesis pass | ~3,000 | ~500 | ~3,500 |
| Chat request | ~5,500 | ~500 | ~6,000 |
| Report generation | ~5,800 | ~2,000 | ~7,800 |

For a 50-page document + 3 chat questions:
- Upload + analysis: ~14,000 tokens
- Chat (3×): ~18,000 tokens
- **Total: ~32,000 tokens**

Free tier on OpenRouter (Qwen): ~200,000 tokens/day. Comfortable for development.

---

## Build Order Checklist

```
Phase 0: Foundation
  [ ] PostgreSQL + pgvector running
  [ ] .env configured
  [ ] /health returns 200

Phase 1: Ingestion
  [ ] PDF/DOCX/TXT parsing works
  [ ] Chunking produces sensible output
  [ ] Document status lifecycle works (pending → processing → ready/error)

Phase 2: Embeddings
  [ ] All chunks get Cohere embeddings
  [ ] pgvector cosine search returns results
  [ ] Reranking works

Phase 3: Carbon Agent
  [ ] Two-pass extraction produces profile_json
  [ ] Dynamic fields captured in additional_fields
  [ ] Hallucination labels (extracted/inferred/null) present

Phase 4: Chat
  [ ] RAG pipeline returns grounded answers
  [ ] Sources array populated
  [ ] Label notation (✅⚠️🔮📋) used

Phase 5: Model Router
  [ ] Task-to-model routing works
  [ ] Fallback triggers on rate limit

Phase 6: Knowledge Base
  [ ] load_knowledge.py script works
  [ ] Knowledge chunks retrieved in chat

Phase 7: Reports
  [ ] /projects/{id}/report returns markdown
  [ ] All 7 report sections present
```

---

## Common Pitfalls

**1. Sending the whole document to the LLM**
Don't. Sample chunks in Pass 1, synthesise in Pass 2.

**2. Wrong Cohere `input_type`**
`search_document` for storing, `search_query` for retrieval. Mixing these silently degrades results.

**3. Background task uses closed DB session**
Background tasks need their own `SessionLocal()` — they can't reuse the request session.

**4. Forgetting to filter `WHERE embedding IS NOT NULL`**
pgvector crashes if you try to compute distance against a NULL vector.

**5. Not validating JSON from LLM**
Always strip markdown fences (```json...```) before `json.loads()`. Always wrap in try/except.

**6. OpenRouter rate limits are per-model, not per-account**
If Qwen is rate-limited, Llama likely isn't. This is why the model router matters.

**7. Cohere batching**
Free tier: max 96 texts per embed call. Batch your chunks; don't call embed() in a loop.

---

## Environment Variables Reference

```bash
# OpenRouter
OPENROUTER_API_KEY=sk-or-...

# Models (override defaults)
MODEL_EXTRACTION=qwen/qwen-2.5-72b-instruct:free
MODEL_CHAT=meta-llama/llama-3.1-70b-instruct:free
MODEL_REPORT=mistralai/mistral-7b-instruct:free
MODEL_FALLBACK=google/gemma-2-9b-it:free

# Cohere
COHERE_API_KEY=...

# Database
DATABASE_URL=postgresql://postgres:password@localhost:5432/carbon_advisor

# Token budgets
MAX_CHUNK_TOKENS=400
MAX_CONTEXT_CHUNKS=6
MAX_COMPLETION_TOKENS=2000

# App
APP_ENV=development
```

---

## First Thing to Build Tomorrow

```bash
# 1. Install postgres and pgvector
brew install postgresql pgvector  # or apt equivalent

# 2. Create the database
createdb carbon_advisor
psql -d carbon_advisor -c "CREATE EXTENSION vector;"

# 3. Create venv and install
python -m venv venv && source venv/bin/activate
pip install fastapi uvicorn sqlalchemy psycopg2-binary pgvector pydantic pydantic-settings python-dotenv

# 4. Create .env, write config.py, database.py, main.py
# 5. Run: uvicorn app.main:app --reload
# 6. Confirm /health works
# 7. Only then move to Phase 1
```

One phase at a time. Don't write the LLM service until the DB is working. Don't write chat until RAG retrieval is tested.

---

*Blueprint version 1.0 — update as you learn what works in your environment.*
