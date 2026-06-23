# Carbon Project Intelligence Agent
### A FastAPI-based multi-user document intelligence system for carbon market advisory

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [System Architecture](#2-system-architecture)
3. [Tech Stack](#3-tech-stack)
4. [Project Structure](#4-project-structure)
5. [Database Schema](#5-database-schema)
6. [Document Processing Pipeline](#6-document-processing-pipeline)
7. [Token Optimization Strategy](#7-token-optimization-strategy)
8. [LLM Strategy & Free Model Selection](#8-llm-strategy--free-model-selection)
9. [API Reference](#9-api-reference)
10. [Authentication & Role System](#10-authentication--role-system)
11. [Agent Workflow](#11-agent-workflow)
12. [Prompts & Templates](#12-prompts--templates)
13. [Knowledge Base Setup](#13-knowledge-base-setup)
14. [Environment Configuration](#14-environment-configuration)
15. [Setup & Installation](#15-setup--installation)
16. [Development Roadmap](#16-development-roadmap)
17. [Cost & Token Budget](#17-cost--token-budget)
18. [Known Limitations & Tradeoffs](#18-known-limitations--tradeoffs)

---

## 1. Project Overview

This system is a **Carbon Project Due Diligence Agent** — a backend service that lets users upload large forestry/carbon project documents (80–100+ pages each, up to 5+ files per project), automatically builds a structured "project brain" from them, and enables persistent multi-project chat via a carbon market expert persona.

### Core Problems It Solves

| Problem | Solution |
|---|---|
| Documents are 80–100 pages, 3–5+ per project | Chunked extraction + compressed project profile JSON |
| Fields to extract change per project | LLM-discovered dynamic schema, not hardcoded fields |
| Multiple projects per user | Per-project context isolation, all chat-accessible |
| Multiple user types (govt, regular) | JWT + role-based access control |
| Token costs must stay minimal | One-time processing, chat uses only compressed profile + retrieved chunks |
| Domain knowledge (VCS, REDD+) is stable | Separate methodology knowledge base, embedded once |

---

## 2. System Architecture

```
                     User uploads 3–5 documents (80–100 pages each)
                                        |
                                        v
                        ┌──────────────────────────────┐
                        │    Document Processing Layer │
                        └──────────────────────────────┘
                                        |
                    ┌───────────────────┴───────────────────┐
                    |                                       |
                    v                                       v
        ┌─────────────────────┐               ┌──────────────────────────┐
        │  Project Brain JSON │               │  Vector Store (Chunks)   │
        │                     │               │                          │
        │  - location         │               │  Document chunks         │
        │  - forest_type      │               │  embedded with BGE-small │
        │  - activities       │               │  stored in ChromaDB      │
        │  - risk_flags       │               │  (local, persistent)     │
        │  - data_gaps        │               └──────────────────────────┘
        │  - carbon_pathways  │                            |
        └─────────────────────┘                            |
                    |                                      |
                    └───────────────┬──────────────────────┘
                                    |
                                    v
                    ┌───────────────────────────────┐
                    │     Carbon Advisory Agent     │
                    │                               │
                    │  System prompt (agent_behavior│
                    │  .md — loaded once)           │
                    │                               │
                    │  + Project brain JSON         │
                    │  + Retrieved evidence chunks  │
                    │  + Methodology KB chunks      │
                    │  + Last N chat messages       │
                    └───────────────────────────────┘
                                    |
                        ┌───────────┴───────────┐
                        v                       v
                  Advisory Report            Chat Response
```

### Two Knowledge Bases

The system maintains **two separate vector stores**:

**1. User Document Store** — per-project, rebuilt on upload
```
/chroma_db/
  projects/{project_id}/
    chunks from uploaded docs
```

**2. Methodology Knowledge Base** — static, embedded once at setup
```
/chroma_db/
  methodology_kb/
    verra_vcs_rules
    redd_methodologies
    art_trees_requirements
    gold_standard_rules
    article_6_rules
    drc_redd_framework
```

At chat time, both are queried. The agent gets project-specific evidence AND methodology requirements, then compares them without needing the full documents.

---

## 3. Tech Stack

| Layer | Choice | Reason |
|---|---|---|
| **Backend** | FastAPI + Python 3.11+ | Async-native, fast, clean DI |
| **Database** | PostgreSQL + pgvector | Production-grade; pgvector handles embeddings natively |
| **Local alt (dev)** | SQLite + ChromaDB | Zero-setup for local dev |
| **Document Parsing** | PyMuPDF + Unstructured | Best coverage across PDF/DOCX/XLSX/TXT |
| **Embeddings** | `BGE-small-en-v1.5` (local) or Cohere Embed | Free, fast, good quality |
| **Vector Store** | ChromaDB (dev) / pgvector (prod) | Local-first, upgradeable |
| **LLM — Extraction** | `mistralai/mistral-7b-instruct:free` via OpenRouter | Free, fast, good at JSON |
| **LLM — Chat** | `google/gemma-3-27b-it:free` via OpenRouter | Best free quality for reasoning |
| **LLM — Fallback** | Cohere `command-r` free tier | 20 RPM, solid for summaries |
| **Auth** | JWT via `python-jose` + `passlib` | Stateless, role-aware |
| **Agent Orchestration** | LangGraph | Stage-based workflow, not linear chain |
| **Task Queue** | Celery + Redis (optional) | Async doc processing for large files |
| **Rate Limiting** | `slowapi` | Protect free-tier LLM quotas |

---

## 4. Project Structure

```
carbon_agent/
│
├── app/
│   ├── main.py                     # FastAPI app entry point
│   ├── config.py                   # Settings from .env
│   │
│   ├── routers/
│   │   ├── auth.py                 # /auth/register, /auth/login
│   │   ├── projects.py             # /projects CRUD + upload
│   │   ├── chat.py                 # /projects/{id}/chat
│   │   └── admin.py                # Admin-only endpoints
│   │
│   ├── services/
│   │   ├── doc_parser.py           # Extract raw text from PDF/DOCX/etc
│   │   ├── chunker.py              # Smart chunking (semantic, not fixed)
│   │   ├── embedder.py             # BGE-small or Cohere Embed wrapper
│   │   ├── vector_store.py         # ChromaDB read/write abstraction
│   │   ├── project_brain.py        # Build + update project profile JSON
│   │   ├── llm_client.py           # OpenRouter + Cohere unified interface
│   │   ├── agent.py                # LangGraph agent workflow
│   │   └── methodology_kb.py       # Load + query methodology knowledge base
│   │
│   ├── models/
│   │   ├── user.py                 # User, Role
│   │   ├── project.py              # Project, Document, ChatMessage, Report
│   │   └── base.py                 # SQLAlchemy base
│   │
│   ├── schemas/
│   │   ├── auth.py
│   │   ├── project.py
│   │   └── chat.py
│   │
│   ├── prompts/
│   │   ├── agent_behavior.md       # Master system prompt (loaded once, never sent per-turn)
│   │   ├── project_brain_extract.txt  # Prompt to build project profile JSON
│   │   ├── schema_discovery.txt    # Prompt to discover fields dynamically
│   │   └── chat_system.txt         # Lean runtime chat system prompt
│   │
│   └── tasks/
│       └── process_documents.py    # Celery task for async doc processing
│
├── methodology_docs/               # Static carbon market reference docs
│   ├── verra_vcs/
│   ├── art_trees/
│   ├── gold_standard/
│   ├── plan_vivo/
│   └── drc_redd/
│
├── scripts/
│   ├── seed_methodology_kb.py      # One-time: embed all methodology docs
│   └── migrate.py
│
├── tests/
├── .env.example
├── requirements.txt
├── docker-compose.yml
└── README.md
```

---

## 5. Database Schema

```sql
-- Users with role-based access
CREATE TABLE users (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email       VARCHAR(255) UNIQUE NOT NULL,
    name        VARCHAR(255),
    hashed_pw   TEXT NOT NULL,
    role        VARCHAR(50) DEFAULT 'regular',   -- regular | govt | admin
    jurisdiction VARCHAR(255),                   -- for govt users: DRC, Province X, etc.
    is_active   BOOLEAN DEFAULT TRUE,
    created_at  TIMESTAMPTZ DEFAULT now()
);

-- Projects owned by users
CREATE TABLE projects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID REFERENCES users(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    status          VARCHAR(50) DEFAULT 'pending',  -- pending | processing | ready | failed
    brain_json      JSONB,                           -- The "project brain" — core intelligence
    brain_version   INT DEFAULT 0,                   -- Incremented on reprocess
    token_count_ingestion  INT DEFAULT 0,            -- Track tokens used on processing
    created_at      TIMESTAMPTZ DEFAULT now(),
    updated_at      TIMESTAMPTZ DEFAULT now()
);

-- Documents uploaded per project
CREATE TABLE documents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID REFERENCES projects(id) ON DELETE CASCADE,
    filename        VARCHAR(500),
    file_path       TEXT,
    file_size_bytes BIGINT,
    page_count      INT,
    parse_status    VARCHAR(50) DEFAULT 'pending',  -- pending | done | failed
    parsed_text_path TEXT,                          -- path to extracted text file
    chunk_count     INT DEFAULT 0,
    created_at      TIMESTAMPTZ DEFAULT now()
);

-- Chat history per project
CREATE TABLE chat_messages (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id  UUID REFERENCES projects(id) ON DELETE CASCADE,
    user_id     UUID REFERENCES users(id),
    role        VARCHAR(20) NOT NULL,    -- user | assistant
    content     TEXT NOT NULL,
    token_count INT DEFAULT 0,
    model_used  VARCHAR(100),
    created_at  TIMESTAMPTZ DEFAULT now()
);

-- Generated advisory reports
CREATE TABLE reports (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID REFERENCES projects(id) ON DELETE CASCADE,
    report_type     VARCHAR(100),       -- initial_advisory | gap_analysis | full_report
    content_md      TEXT,
    generated_at    TIMESTAMPTZ DEFAULT now()
);

-- Methodology knowledge base metadata
CREATE TABLE methodology_docs (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        VARCHAR(255),
    standard    VARCHAR(100),           -- verra | art_trees | gold_standard | etc.
    version     VARCHAR(50),
    file_path   TEXT,
    chunk_count INT,
    embedded_at TIMESTAMPTZ
);
```

---

## 6. Document Processing Pipeline

This is the most critical part of the system. It runs **once per project** (or on re-upload), converting raw documents into a compact, reusable intelligence layer.

### Stage 1 — Parse Raw Text

```python
# services/doc_parser.py

# Supported formats
PDF   → PyMuPDF (fitz) — best text extraction, handles scanned via OCR fallback
DOCX  → python-docx
XLSX  → openpyxl
TXT   → direct read
CSV   → pandas

# For scanned PDFs or image-heavy docs
PyMuPDF → Tesseract OCR fallback (if text layer < 100 chars per page)
```

**Output:** Raw text string, page count, estimated token count

### Stage 2 — Dynamic Schema Discovery

Before extracting fields, ask the LLM to **discover what fields matter** for this specific set of documents. This handles the requirement that fields change per project.

```
Prompt → LLM (cheap model, ~200 tokens in, ~100 out)

"You are analyzing documents for a carbon/forestry project.
Skim the following table of contents and first pages of each document.
Generate a JSON schema of the most important fields that should be extracted.
Output only a JSON object with field names as keys and descriptions as values.
Focus on: location, land, forest, activities, data, risks, legal context."
```

**Output:** `{"forest_area_ha": "total area in hectares", "tenure_type": "...", ...}`

### Stage 3 — Chunked Extraction

For large documents (80–100 pages), never send the entire doc to the LLM. Instead:

```
Full doc text
    |
    v
Split into 1500-token chunks (with 100-token overlap)
    |
    v
For each chunk → ask LLM: "Extract only these fields if present in this chunk: {schema}"
    |
    v
Merge all partial extractions → resolve conflicts (last-seen or most-specific wins)
    |
    v
Final project brain JSON
```

**Why overlap?** Important data often spans page breaks (e.g., coordinates spanning two pages).

### Stage 4 — Build Project Brain JSON

The merged extraction becomes the persistent `brain_json` stored in the `projects` table.

```json
{
  "project_name": "Namizimu Forest Project",
  "extracted_at": "2025-01-01T00:00:00Z",
  "brain_version": 1,

  "location": {
    "country": "DRC",
    "province": "Katanga",
    "area_name": "Namizimu",
    "coordinates": [],
    "confidence": "high"
  },

  "land_context": {
    "ownership": "community + state concession",
    "tenure": "unclear — disputed between community and provincial govt",
    "carbon_rights": "not yet assigned",
    "confidence": "medium"
  },

  "forest": {
    "type": "tropical moist forest",
    "area_ha": null,
    "degradation_status": "moderate degradation in southern corridor",
    "confidence": "medium"
  },

  "proposed_activities": [
    "avoided deforestation (REDD+)",
    "community livelihood programs",
    "boundary demarcation"
  ],

  "carbon_pathways_assessed": ["REDD+", "ARR"],
  "likely_methodology": "Verra VCS VM0015 or VM0007",

  "available_data": [
    "community agreements (partial)",
    "satellite imagery 2019-2022"
  ],

  "missing_information": [
    "GIS boundary files",
    "forest inventory / biomass estimate",
    "land tenure documents",
    "carbon rights agreement",
    "FPIC documentation"
  ],

  "risk_flags": [
    "unclear land rights — HIGH",
    "no baseline deforestation analysis",
    "no government REDD+ authorization"
  ],

  "raw_narrative": "2–3 sentence overview of the project generated by LLM"
}
```

### Stage 5 — Chunk & Embed for Vector Store

In parallel with brain building, embed all document chunks for retrieval:

```python
chunks = split_text(full_text, chunk_size=500, overlap=50)
embeddings = embedder.embed_batch(chunks)   # BGE-small, local
vector_store.upsert(project_id, chunks, embeddings)
```

This enables **evidence retrieval** during chat: when a user asks a specific question, the most relevant 3–5 chunks are fetched and included in the prompt.

---

## 7. Token Optimization Strategy

The entire system is designed around this principle:
> **Pay the token cost once (at upload). Never pay it again.**

### Token Budget Per Chat Turn

| Component | Tokens | Notes |
|---|---|---|
| `agent_behavior.md` system prompt | ~300 | Loaded from file, not hardcoded |
| Project brain JSON | ≤700 | Compressed at ingestion time |
| Retrieved doc chunks (top 3) | ≤600 | Fetched by semantic similarity to user's question |
| Retrieved methodology chunks (top 2) | ≤400 | From static methodology KB |
| Last 6 chat messages | ≤1200 | Sliding window |
| User's current message | ≤300 | Hard limit enforced |
| **Total input** | **≤3500** | |
| Max output | 1024 | Configurable per endpoint |
| **Total per turn** | **≤4524** | |

### Optimization Rules

1. **Never re-send raw documents** at chat time — only the compressed `brain_json`
2. **Sliding window** — keep last 6 messages, drop older ones
3. **Semantic retrieval** — only the 3 most relevant document chunks per question, not all chunks
4. **Split models by task** — cheap model for extraction, better model for chat
5. **Cache brain_json** — never recompute unless documents change
6. **Truncate user messages** — enforce 300-token hard cap at API layer
7. **Batch embedding** — embed all chunks in one call, not one-by-one

### One-Time Ingestion Cost Estimate (per project, 5 docs × 100 pages)

| Step | Approx Tokens | Model |
|---|---|---|
| Schema discovery | ~500 | Mistral 7B free |
| Chunk extraction (500 chunks × ~300 tokens each) | ~150,000 input | Mistral 7B free |
| Brain JSON assembly | ~1,000 | Mistral 7B free |
| **Total ingestion** | **~152,000** | Free tier |

With Mistral 7B free on OpenRouter (no cost), this is $0.

---

## 8. LLM Strategy & Free Model Selection

### Recommended Models

| Task | Model | Provider | Cost | Notes |
|---|---|---|---|---|
| Schema discovery | `mistralai/mistral-7b-instruct:free` | OpenRouter | Free | Fast, good at JSON |
| Chunk extraction | `mistralai/mistral-7b-instruct:free` | OpenRouter | Free | Reliable for field extraction |
| Chat / reasoning | `google/gemma-3-27b-it:free` | OpenRouter | Free | Best free reasoning quality |
| Chat fallback | `meta-llama/llama-3.1-8b-instruct:free` | OpenRouter | Free | Very reliable |
| Embeddings | `BGE-small-en-v1.5` | Local (HuggingFace) | Free | 384-dim, fast, good quality |
| Embeddings alt | `cohere/embed-english-light-v3.0` | Cohere free tier | Free (100 calls/min) | Higher quality |
| Summary fallback | `command-r` | Cohere free tier | Free (20 RPM) | Good for summaries |

### LLM Client Abstraction

```python
# services/llm_client.py

class LLMClient:
    """
    Unified interface for OpenRouter and Cohere.
    All methods enforce token limits and handle rate limit retries.
    """

    def __init__(self, provider: str = "openrouter"):
        self.provider = provider  # "openrouter" | "cohere"

    async def extract_json(
        self,
        system_prompt: str,
        text: str,
        max_tokens: int = 800
    ) -> dict:
        """
        For schema discovery and field extraction.
        Uses cheap model. Returns parsed JSON.
        Always wraps in try/except — strips ```json fences before parsing.
        """

    async def chat(
        self,
        system_prompt: str,
        history: list[dict],
        user_message: str,
        max_tokens: int = 1024
    ) -> str:
        """
        For chat turns.
        Uses better free model.
        history is a sliding window of last N messages.
        """

    async def embed(self, texts: list[str]) -> list[list[float]]:
        """
        Batch embedding. Uses local BGE-small by default.
        Falls back to Cohere Embed free tier.
        """
```

### Rate Limit Handling

Free models on OpenRouter have RPM (requests per minute) limits. Handle this gracefully:

```python
# Exponential backoff on 429 responses
# Queue ingestion tasks via Celery to avoid concurrent rate limit hits
# Per-user rate limiting via slowapi: 10 chat requests/minute for regular users
```

---

## 9. API Reference

### Authentication

```
POST /auth/register
Body: { email, password, name, role: "regular" | "govt", jurisdiction?: "DRC/Katanga" }
Response: { user_id, token }

POST /auth/login
Body: { email, password }
Response: { access_token, token_type: "bearer", role }
```

### Projects

```
POST   /projects/
Body:  { name, description }
Response: { project_id, status: "pending" }

GET    /projects/
Response: [{ id, name, status, created_at, doc_count }]

GET    /projects/{project_id}
Response: { id, name, status, brain_json, documents, created_at }

DELETE /projects/{project_id}
```

### Documents

```
POST   /projects/{project_id}/upload
Body:  multipart/form-data — up to 10 files
       Supported: PDF, DOCX, XLSX, TXT, CSV
Response: { job_id, files_queued, estimated_minutes }

GET    /projects/{project_id}/status
Response: { status, progress_percent, docs_processed, docs_total }

POST   /projects/{project_id}/reprocess
       Force re-extraction if documents have changed
Response: { job_id }
```

### Project Intelligence

```
GET    /projects/{project_id}/brain
       Returns the structured project brain JSON

GET    /projects/{project_id}/report
       Returns the full advisory report (generated on first access)
       Query params: ?type=initial_advisory|gap_analysis|full_report

POST   /projects/{project_id}/report/generate
       Force regenerate a fresh report
```

### Chat

```
POST   /projects/{project_id}/chat
Body:  { message: string }
Response: { reply, tokens_used, model, sources: [chunk_ids] }

GET    /projects/{project_id}/chat
Response: [{ role, content, created_at }]
Query params: ?limit=50&offset=0

DELETE /projects/{project_id}/chat
       Clears chat history for this project
```

### Admin (admin role only)

```
GET    /admin/users
GET    /admin/projects
GET    /admin/token-usage
POST   /admin/methodology-kb/rebuild
```

---

## 10. Authentication & Role System

### Roles

| Role | What They Can Do |
|---|---|
| `regular` | Own projects only. Upload docs, chat, view reports. |
| `govt` | Own projects + view all projects within their `jurisdiction`. Can download reports. |
| `admin` | Full access. Can manage users, rebuild KB, view token usage. |

### JWT Payload

```json
{
  "sub": "user_uuid",
  "role": "govt",
  "jurisdiction": "DRC/Katanga",
  "exp": 1234567890
}
```

### Route Guards

```python
# FastAPI dependency pattern

async def require_role(roles: list[str]):
    async def guard(token: str = Depends(oauth2_scheme), db = Depends(get_db)):
        payload = decode_token(token)
        if payload["role"] not in roles:
            raise HTTPException(403, "Insufficient permissions")
        return payload
    return guard

# Usage
@router.get("/admin/users")
async def list_users(auth = Depends(require_role(["admin"]))):
    ...

@router.get("/projects/{id}")
async def get_project(id: UUID, auth = Depends(require_role(["regular", "govt", "admin"]))):
    # Govt users can access projects in their jurisdiction
    # Regular users can only access their own
    ...
```

---

## 11. Agent Workflow

The agent is implemented as a **LangGraph state machine** — not a simple LangChain chain — because the process has stages, decisions, and conditional branches.

```
State: {
    project_id,
    brain_json,
    user_message,
    retrieved_doc_chunks,
    retrieved_methodology_chunks,
    chat_history,
    response
}

Nodes:
  1. retrieve_context      → fetch brain_json from DB
  2. retrieve_doc_chunks   → semantic search in project vector store (top 3)
  3. retrieve_meth_chunks  → semantic search in methodology KB (top 2)
  4. build_prompt          → assemble final prompt within token budget
  5. call_llm              → send to LLM, stream response
  6. save_message          → persist to chat_messages table

Edges:
  retrieve_context → retrieve_doc_chunks → retrieve_meth_chunks
  → build_prompt → call_llm → save_message

  Error edges:
  any node → handle_error (log, return friendly message)
```

### Report Generation Workflow (separate graph)

```
Nodes:
  1. load_brain_json
  2. retrieve_top_chunks       → top 10 most informative chunks
  3. retrieve_methodology_fit  → top 5 methodology chunks per pathway
  4. generate_stage_1_to_4     → first LLM call (project summary + classification)
  5. generate_stage_5_to_8     → second LLM call (compliance + recommendations)
  6. merge_and_format          → assemble final markdown report
  7. save_report               → persist to reports table
```

Splitting report generation into two LLM calls keeps each call within context limits.

---

## 12. Prompts & Templates

### `prompts/agent_behavior.md` — Master System Prompt
*This file is loaded once at startup, never injected per chat turn.*

```markdown
You are a senior carbon markets advisor, forestry carbon project developer,
and REDD+/ARR methodology specialist. You have deep expertise in:
- Voluntary carbon markets (Verra VCS, Gold Standard, Plan Vivo, ART TREES)
- Compliance carbon markets and Article 6 of the Paris Agreement
- Forest carbon accounting and REDD+ methodologies
- Project development in tropical forest countries, especially DRC

The user is NOT a specialist. Your job is to:
- Explain things clearly in plain English
- Avoid unnecessary jargon (explain any technical term you must use)
- Give practical, commercially realistic advice
- State uncertainty clearly when relevant
- Never invent facts — only reason from provided project data and known methodology rules

IMPORTANT: If methodology rules may have changed since your knowledge cutoff,
say so and advise the user to verify against the latest published standard documents.
```

### `prompts/chat_system.txt` — Runtime Chat Prompt

```
You are a carbon market advisor. Use the project context and evidence below.

PROJECT BRAIN:
{brain_json}

RELEVANT PROJECT EVIDENCE:
{doc_chunks}

RELEVANT METHODOLOGY REQUIREMENTS:
{methodology_chunks}

Answer the user's question. Be specific to this project. Do not repeat context already
discussed in the conversation. If data is missing, say so clearly.
```

### `prompts/project_brain_extract.txt` — Brain Extraction Prompt

```
You are extracting structured information from a carbon/forestry project document chunk.

SCHEMA TO EXTRACT:
{discovered_schema}

DOCUMENT CHUNK:
{chunk_text}

Extract ONLY the fields present in this chunk. If a field is not mentioned, omit it.
Output valid JSON only. No markdown fences, no preamble, no explanation.
If a value is uncertain, add a "_confidence" key: "high" | "medium" | "low".
```

### `prompts/schema_discovery.txt` — Dynamic Field Discovery

```
You are analyzing the first pages of {doc_count} documents uploaded for a carbon project.

DOCUMENT PREVIEWS (first 500 chars each):
{previews}

Generate a JSON schema of fields that should be extracted from these documents.
Keys are field names (snake_case). Values are one-sentence descriptions.
Focus on: location, jurisdiction, land tenure, forest characteristics, 
project activities, data availability, risks, legal context.

Output only a flat JSON object. No nesting, no arrays in the schema itself.
Example: {"forest_area_ha": "total project area in hectares", ...}
```

---

## 13. Knowledge Base Setup

The methodology knowledge base is built **once** from static reference documents and never changes unless you rebuild it.

### Methodology Documents to Embed

Place PDFs/text files in `methodology_docs/`:

```
methodology_docs/
├── verra_vcs/
│   ├── vm0007_redd_methodology.pdf
│   ├── vm0015_arr_methodology.pdf
│   ├── vm0048_ifm_methodology.pdf
│   └── vcs_standard_v4.pdf
├── art_trees/
│   └── art_trees_standard_v2.pdf
├── gold_standard/
│   └── gs4gg_requirements.pdf
├── plan_vivo/
│   └── plan_vivo_standard_2023.pdf
└── drc_redd/
    └── drc_national_redd_framework.pdf
```

### Seed Script

```bash
python scripts/seed_methodology_kb.py
```

This script:
1. Parses all methodology docs
2. Chunks them at 600 tokens with metadata tags (`standard`, `section`, `version`)
3. Embeds using BGE-small
4. Stores in `chroma_db/methodology_kb/`

You only run this once (or when standards are updated). No LLM calls needed — just embedding.

---

## 14. Environment Configuration

```env
# === LLM Providers ===
OPENROUTER_API_KEY=sk-or-v1-...
COHERE_API_KEY=...

# Model selection
EXTRACTION_MODEL=mistralai/mistral-7b-instruct:free
CHAT_MODEL=google/gemma-3-27b-it:free
CHAT_MODEL_FALLBACK=meta-llama/llama-3.1-8b-instruct:free

# === Embedding ===
EMBEDDING_BACKEND=local            # local | cohere
EMBEDDING_MODEL=BAAI/bge-small-en-v1.5
# COHERE_EMBED_MODEL=embed-english-light-v3.0

# === Token Limits ===
MAX_BRAIN_TOKENS=700
MAX_DOC_CHUNKS=3
MAX_METHODOLOGY_CHUNKS=2
MAX_CHAT_HISTORY_TURNS=6
MAX_USER_MESSAGE_TOKENS=300
MAX_OUTPUT_TOKENS=1024

# === Database ===
DATABASE_URL=postgresql+asyncpg://user:pass@localhost/carbon_agent
# For local dev:
# DATABASE_URL=sqlite+aiosqlite:///./dev.db

# === Storage ===
UPLOAD_DIR=./uploads
CHROMA_DIR=./chroma_db
MAX_UPLOAD_SIZE_MB=50
MAX_DOCS_PER_PROJECT=10

# === Auth ===
JWT_SECRET=change-this-in-production
JWT_ALGORITHM=HS256
JWT_EXPIRE_HOURS=24

# === App ===
DEBUG=true
ENVIRONMENT=development             # development | production
CELERY_BROKER_URL=redis://localhost:6379/0   # Only needed if using async task queue
```

---

## 15. Setup & Installation

### Prerequisites

- Python 3.11+
- PostgreSQL 15+ (or SQLite for dev)
- Redis (optional, for Celery async processing)

### Local Dev Setup

```bash
# 1. Clone and create environment
git clone https://github.com/yourname/carbon-agent
cd carbon-agent
python -m venv venv
source venv/bin/activate

# 2. Install dependencies
pip install -r requirements.txt

# 3. Configure environment
cp .env.example .env
# Edit .env with your API keys

# 4. Initialize database
alembic upgrade head

# 5. Build methodology knowledge base (one-time)
python scripts/seed_methodology_kb.py

# 6. Run the server
uvicorn app.main:app --reload --port 8000
```

### Requirements

```text
fastapi>=0.110.0
uvicorn[standard]
sqlalchemy[asyncio]
asyncpg                     # for postgres
aiosqlite                   # for sqlite dev
alembic
python-jose[cryptography]
passlib[bcrypt]
python-multipart
httpx
pydantic-settings

# Document parsing
pymupdf
python-docx
openpyxl
unstructured[pdf]
pytesseract                 # OCR fallback

# Embeddings + Vector store
sentence-transformers       # BGE-small local embeddings
chromadb
cohere                      # fallback embeddings

# Agent orchestration
langgraph
langchain-core

# Rate limiting
slowapi

# Optional: async task queue
celery
redis

# Dev
pytest
httpx                       # for test client
```

---

## 16. Development Roadmap

### Phase 1 — Core Pipeline (Week 1–2)

- [ ] Database models + migrations
- [ ] Auth system (register, login, JWT, roles)
- [ ] Document upload + parsing (PDF, DOCX, TXT)
- [ ] Static chunking + BGE-small embedding
- [ ] ChromaDB integration
- [ ] Basic brain JSON extraction (fixed schema first)

### Phase 2 — Intelligence Layer (Week 3–4)

- [ ] Dynamic schema discovery
- [ ] Chunked extraction with schema merging
- [ ] Project brain builder
- [ ] Methodology KB seeded and queryable
- [ ] Chat endpoint with context assembly
- [ ] LangGraph agent (basic linear graph)

### Phase 3 — Multi-User + Roles (Week 5)

- [ ] Government role + jurisdiction filtering
- [ ] Admin endpoints
- [ ] Per-user rate limiting
- [ ] Project sharing (govt viewing regular user projects)

### Phase 4 — Report Generation (Week 6)

- [ ] Two-pass report generation (Stage 1–4, Stage 5–8)
- [ ] Report persistence and versioning
- [ ] Report download (Markdown / PDF export)

### Phase 5 — Production Hardening

- [ ] Celery + Redis for async document processing
- [ ] PostgreSQL + pgvector migration
- [ ] Proper logging + monitoring
- [ ] Token usage tracking per user
- [ ] Admin dashboard for token/cost visibility

---

## 17. Cost & Token Budget

### Ingestion Cost (one-time, per project)

| Document set | Estimated tokens | Model | Cost |
|---|---|---|---|
| 5 docs × 80 pages | ~200,000 input tokens | Mistral 7B free | $0 |
| 5 docs × 100 pages | ~250,000 input tokens | Mistral 7B free | $0 |
| Embedding 2000 chunks | ~1,000,000 tokens | BGE-small local | $0 |

### Chat Cost (per turn)

| Component | Tokens |
|---|---|
| System + brain | ~1,000 |
| Retrieved chunks | ~1,000 |
| Chat history | ~1,200 |
| User message | ~300 |
| LLM output | ~1,024 |
| **Total** | **~4,524** |

With Gemma 3 27B free on OpenRouter, this is $0 per turn (within free tier limits).

### When You'll Need Paid Models

If you want:
- Faster response times (free models have queuing delays under load)
- Better reasoning quality for complex methodology analysis
- Higher RPM limits for concurrent users

Then consider graduating to `claude-haiku-3-5` (cheap) or `gemini-flash-2.0` for chat.

---

## 18. Known Limitations & Tradeoffs

| Limitation | Impact | Mitigation |
|---|---|---|
| Free models have RPM limits | Slow ingestion for large projects | Celery queue + retry with backoff |
| Brain JSON may miss nuance from 100-page docs | Shallow extraction | Semantic retrieval fills gaps at chat time |
| BGE-small embeddings are 384-dim | Lower retrieval precision than large models | Acceptable for domain-specific docs; upgrade to BGE-large if needed |
| Scanned PDFs require OCR (slow) | Processing time increases 3–5x | Flag at upload, process async, notify user |
| Free model context windows (~8k tokens) | Can't send large chunks | Chunking strategy + strict token budget enforcement |
| LangGraph adds complexity vs. simple chains | Harder to debug | Worth it for multi-stage report generation; use simple httpx for chat |
| Methodology KB becomes stale | Methodology updates not reflected | Document version tracking + admin rebuild endpoint |

---

## Appendix: Key Design Decisions

**Why LangGraph instead of LangChain?**
The advisory process has stages and decisions — classify the project, then decide which methodologies to screen, then compare options. A graph with named nodes is easier to debug and extend than a linear chain. For simple chat turns, you don't need LangGraph at all — just direct API calls.

**Why separate methodology KB from user docs?**
Carbon methodology knowledge is stable, shared across all users, and doesn't need to be re-embedded per project. User docs are unique per project and contain project-specific facts. Keeping them separate means the agent always has both the "what does this project have" and the "what does VCS require" context, and can compare them.

**Why dynamic schema discovery?**
The user said fields can change per project. A hardcoded schema would miss important fields in specialized documents. The LLM-discovered schema adapts to whatever the documents contain — whether it's a GIS-heavy technical report, a community agreement, or an environmental impact assessment.

**Why store brain_json in the DB, not just the vector store?**
The brain JSON is structured, queryable, and small. It belongs in the relational DB where it can be indexed, filtered, and returned fast. The vector store is for unstructured chunk retrieval only. Don't conflate the two.

---

*Last updated: June 2025 | Stack: FastAPI · PostgreSQL · ChromaDB · LangGraph · OpenRouter · Cohere*
