# Carbon Markets Advisor — Local-First AI

An AI-powered backend that ingests forestry carbon project documents (PDF, DOCX, TXT) and answers advisory questions using RAG.

---

## Stack

| Layer | Technology |
|---|---|
| API | FastAPI + Python 3.11+ |
| Database | PostgreSQL + pgvector |
| Embeddings | Cohere `embed-english-v3.0` |
| Reranking | Cohere `rerank-english-v3.0` |
| LLM | Qwen 2.5 72B via OpenRouter (free tier) |
| Doc parsing | PyMuPDF (PDF), python-docx (DOCX) |

---

## Quick Start

### 1. Prerequisites

- Python 3.11+
- PostgreSQL 15+ with pgvector extension
- Free API keys: [OpenRouter](https://openrouter.ai) + [Cohere](https://cohere.com)

### 2. Install pgvector

```bash
# Ubuntu/Debian
sudo apt install postgresql-15-pgvector

# macOS
brew install pgvector
```

### 3. Create database

```sql
CREATE DATABASE carbon_advisor;
```

### 4. Clone and configure

```bash
git clone <repo>
cd carbon-advisor

cp .env.example .env
# Edit .env — fill in your API keys and DATABASE_URL
```

### 5. Install dependencies

```bash
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

### 6. Run the API

```bash
uvicorn app.main:app --reload --port 8000
```

Tables are created automatically on first run.

### 7. (Optional) Load knowledge base

Place methodology PDFs in the `knowledge/` subfolders, then:

```bash
python -m scripts.load_knowledge
```

---

## API Usage

### Create a project

```bash
curl -X POST http://localhost:8000/projects \
  -H "Content-Type: application/json" \
  -d '{"name": "Namizimu DRC Forest", "description": "REDD+ candidate site"}'
```

### Upload a document

```bash
curl -X POST http://localhost:8000/documents/upload \
  -F "project_id=<project_id>" \
  -F "file=@my_project_report.pdf"
```

### Check processing status

```bash
curl http://localhost:8000/documents/<document_id>
```

### Get project profile (structured extraction)

```bash
curl http://localhost:8000/projects/<project_id>
```

### Chat with your documents

```bash
curl -X POST http://localhost:8000/chat \
  -H "Content-Type: application/json" \
  -d '{"project_id": "<id>", "message": "What methodology fits this project?"}'
```

### Generate advisory report

```bash
curl -X POST http://localhost:8000/projects/<project_id>/report
```

---

## Swagger UI

Visit `http://localhost:8000/docs` for interactive API docs.

---

## Token Budget

The system is designed to work within free-tier limits:

| Setting | Default | Purpose |
|---|---|---|
| `MAX_CHUNK_TOKENS` | 400 | Size of each stored chunk |
| `MAX_CONTEXT_CHUNKS` | 6 | Chunks sent to LLM per request |
| `MAX_COMPLETION_TOKENS` | 2000 | Max LLM output per call |

A typical chat call uses ~4–5k tokens total. Adjust in `.env`.

---

## Project Structure

```
app/
├── main.py                  ← FastAPI app + startup
├── core/
│   ├── config.py            ← Settings (env vars)
│   └── security.py          ← API key auth
├── api/routes/
│   ├── documents.py         ← Upload + status
│   ├── projects.py          ← Create, profile, report
│   └── chat.py              ← RAG chat
├── db/
│   ├── database.py          ← Engine + session
│   └── models.py            ← ORM models (pgvector)
├── schemas/                 ← Pydantic request/response
├── services/
│   ├── document_processor.py   ← Text chunking
│   ├── embedding_service.py    ← Cohere embeddings
│   ├── llm_service.py          ← OpenRouter/Qwen
│   ├── retrieval_service.py    ← pgvector search + rerank
│   ├── project_analyzer.py     ← Two-pass LLM extraction
│   └── report_generator.py     ← Advisory report
├── agents/
│   └── carbon_agent.py      ← Full pipeline orchestrator
├── prompts/                 ← All LLM prompt templates
└── utils/
    └── file_parser.py       ← PDF/DOCX/TXT extraction

knowledge/                   ← Methodology reference docs
├── verra/
├── art_trees/
├── gold_standard/
├── article6/
└── drc_redd/

scripts/
└── load_knowledge.py        ← Seed the knowledge base
```

---

## Adding More Agents

The `carbon_agent.py` is Agent 1. To add Agent 2 (e.g. `compliance_agent`):

1. Create `app/agents/compliance_agent.py`
2. Add a new router `app/api/routes/compliance.py`
3. Add prompts in `app/prompts/`
4. Wire it into `main.py`

Each agent follows the same pattern: retrieve → prompt → respond.
