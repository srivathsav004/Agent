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
              
### Two Knowledge Bases

The system maintains **two separate vector stores**:

**1. User Document Store** — per-project, rebuilt on upload
- Located under `/chroma_db/projects/{project_id}/`
- Contains chunks from uploaded docs

**2. Methodology Knowledge Base** — static, embedded once at setup
- Located under `/chroma_db/methodology_kb/`
- Contains reference materials for verra_vcs_rules, redd_methodologies, art_trees_requirements, gold_standard_rules, article_6_rules, and drc_redd_framework

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

The project follows a modular FastAPI layout with separate directories for routers, services, models, schemas, prompts, and tasks. Key components include:

- **Routers**: Authentication, project management, chat, and admin endpoints
- **Services**: Document parsing, chunking, embedding, vector storage, project brain construction, LLM client abstraction, agent workflow, and methodology knowledge base management
- **Models**: SQLAlchemy models for users, projects, documents, chat messages, reports, and methodology documents
- **Schemas**: Pydantic schemas for request/response validation
- **Prompts**: Master system prompt, project brain extraction prompt, schema discovery prompt, and lean runtime chat prompt
- **Tasks**: Celery task definitions for asynchronous document processing
- **Methodology Docs**: Static carbon market reference documents organized by standard (Verra VCS, ART TREES, Gold Standard, Plan Vivo, DRC REDD)

---

## 5. Database Schema

The relational database centers on four core entities:

**Users** — Stores account credentials, name, role (regular/govt/admin), jurisdiction (for government users), and active status.

**Projects** — Owned by users, tracks project name, description, processing status, the compressed "brain" JSON representation, version tracking for brain updates, and token consumption metrics during ingestion.

**Documents** — Tracks uploaded files per project including filename, storage path, file size, page count, parse status, path to extracted text, and chunk count.

**Chat Messages** — Persists conversation history per project with role attribution, content, token count, and model used.

**Reports** — Stores generated advisory reports with type classification, markdown content, and generation timestamp.

**Methodology Docs** — Metadata for the static knowledge base including standard name, version, file path, chunk count, and embedding timestamp.

---

## 6. Document Processing Pipeline

This is the most critical part of the system. It runs **once per project** (or on re-upload), converting raw documents into a compact, reusable intelligence layer.

### Stage 1 — Parse Raw Text

Supported formats include PDF (via PyMuPDF with Tesseract OCR fallback for scanned documents), DOCX, XLSX, TXT, and CSV. The output is raw text string, page count, and estimated token count.

### Stage 2 — Dynamic Schema Discovery

Before extracting fields, the LLM discovers what fields matter for this specific set of documents. This handles the requirement that fields change per project. The LLM skims table of contents and first pages, then generates a JSON schema of the most important fields to extract, focusing on location, land, forest, activities, data, risks, and legal context.

### Stage 3 — Chunked Extraction

For large documents (80–100 pages), the system never sends the entire document to the LLM. Instead, the full text is split into 1500-token chunks with 100-token overlap. Each chunk is processed individually to extract only the relevant schema fields. All partial extractions are then merged, with conflicts resolved using last-seen or most-specific-wins strategies. Overlap is important because important data often spans page breaks (e.g., coordinates spanning two pages).

### Stage 4 — Build Project Brain JSON

The merged extraction becomes the persistent `brain_json` stored in the projects table. This structured representation includes:

- Project identification (name, extraction timestamp, version)
- Location data (country, province, area, coordinates, confidence level)
- Land context (ownership, tenure, carbon rights, confidence)
- Forest characteristics (type, area, degradation status, confidence)
- Proposed activities (avoided deforestation, community programs, etc.)
- Carbon pathways assessed and likely methodology
- Available data sources
- Missing information gaps
- Risk flags with severity levels
- A brief narrative overview generated by the LLM

### Stage 5 — Chunk & Embed for Vector Store

In parallel with brain building, all document chunks are embedded using BGE-small (local) and stored in the vector store. This enables evidence retrieval during chat: when a user asks a specific question, the most relevant chunks are fetched and included in the prompt.

---

## 7. Token Optimization Strategy

The entire system is designed around this principle:
> **Pay the token cost once (at upload). Never pay it again.**

### Token Budget Per Chat Turn

| Component | Tokens | Notes |
|---|---|---|
| Agent behavior system prompt | ~300 | Loaded from file, not hardcoded |
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

The system uses a unified interface for OpenRouter and Cohere providers. All methods enforce token limits and handle rate limit retries with exponential backoff. The client supports three operations: JSON extraction (for schema discovery and field extraction), chat completion (for conversational turns), and batch embedding (using local BGE-small by default, with Cohere fallback).

### Rate Limit Handling

Free models on OpenRouter have RPM (requests per minute) limits. The system handles this gracefully through exponential backoff on 429 responses, queuing ingestion tasks via Celery to avoid concurrent rate limit hits, and per-user rate limiting via slowapi (10 chat requests per minute for regular users).

---

## 9. API Reference

### Authentication

- **Register**: Create account with email, password, name, role (regular or government), and optional jurisdiction
- **Login**: Returns access token, token type, and role

### Projects

- **Create**: Initialize a new project with name and description, returns pending status
- **List**: Retrieve all projects for the authenticated user with status and document counts
- **Get**: Retrieve full project details including brain JSON and associated documents
- **Delete**: Remove project and all associated data

### Documents

- **Upload**: Submit up to 10 files (PDF, DOCX, XLSX, TXT, CSV) per project, returns job ID and estimated processing time
- **Status**: Check processing progress with percentage completion and document counts
- **Reprocess**: Force re-extraction when documents have changed

### Project Intelligence

- **Brain**: Retrieve the structured project brain JSON
- **Report**: Retrieve generated advisory report (initial advisory, gap analysis, or full report), generated on first access
- **Generate Report**: Force regeneration of a fresh report

### Chat

- **Send Message**: Submit a question, receives reply with token usage, model name, and source chunk references
- **History**: Retrieve past messages with pagination support
- **Clear**: Delete all chat history for the project

### Admin (admin role only)

- List all users and projects
- View system-wide token usage statistics
- Rebuild methodology knowledge base

---

## 10. Authentication & Role System

### Roles

| Role | What They Can Do |
|---|---|
| `regular` | Own projects only. Upload docs, chat, view reports. |
| `govt` | Own projects + view all projects within their `jurisdiction`. Can download reports. |
| `admin` | Full access. Can manage users, rebuild KB, view token usage. |

### JWT Payload

The JWT token contains the user UUID, role, jurisdiction (for government users), and expiration timestamp.

### Route Guards

Access control uses FastAPI dependency injection. Government users can access projects in their jurisdiction, regular users can only access their own projects, and administrators have unrestricted access.

---

## 11. Agent Workflow

The agent is implemented as a **LangGraph state machine** — not a simple chain — because the process has stages, decisions, and conditional branches.

### Chat Workflow States

1. **Retrieve Context** — Fetch brain_json from database
2. **Retrieve Document Chunks** — Semantic search in project vector store (top 3)
3. **Retrieve Methodology Chunks** — Semantic search in methodology KB (top 2)
4. **Build Prompt** — Assemble final prompt within token budget
5. **Call LLM** — Send to language model, stream response
6. **Save Message** — Persist to chat history table

Error handling routes any node failure to a dedicated error handler that logs the issue and returns a friendly message.

### Report Generation Workflow (separate graph)

1. **Load Brain JSON**
2. **Retrieve Top Chunks** — Top 10 most informative chunks
3. **Retrieve Methodology Fit** — Top 5 methodology chunks per pathway
4. **Generate Stage 1–4** — First LLM call for project summary and classification
5. **Generate Stage 5–8** — Second LLM call for compliance and recommendations
6. **Merge and Format** — Assemble final markdown report
7. **Save Report** — Persist to reports table

Splitting report generation into two LLM calls keeps each call within context limits.

---

## 12. Prompts & Templates

### Master System Prompt

The agent behavior prompt is loaded once at startup and never injected per chat turn. It establishes the persona as a senior carbon markets advisor, forestry carbon project developer, and REDD+/ARR methodology specialist with expertise in voluntary carbon markets, compliance markets, forest carbon accounting, and tropical forest project development. The advisor explains things clearly in plain English, avoids unnecessary jargon, gives practical commercially realistic advice, states uncertainty clearly, and never invents facts — only reasoning from provided project data and known methodology rules.

### Runtime Chat Prompt

The lean runtime prompt combines the project brain JSON, relevant project evidence chunks, and relevant methodology requirement chunks. The agent answers the user's question specifically to this project, does not repeat already-discussed context, and clearly states when data is missing.

### Brain Extraction Prompt

Used for extracting structured information from document chunks. The LLM receives the discovered schema and document chunk, then extracts only the fields present in that chunk. If a field is not mentioned, it is omitted. Confidence levels (high, medium, low) are added when values are uncertain.

### Schema Discovery Prompt

Analyzes the first pages of uploaded documents to generate a JSON schema of fields that should be extracted. Keys are field names in snake_case, values are one-sentence descriptions. The focus is on location, jurisdiction, land tenure, forest characteristics, project activities, data availability, risks, and legal context.

---

## 13. Knowledge Base Setup

The methodology knowledge base is built **once** from static reference documents and never changes unless explicitly rebuilt.

### Methodology Documents to Embed

Reference materials are organized by standard:
- **Verra VCS**: REDD methodology, ARR methodology, IFM methodology, and VCS standard
- **ART TREES**: TREES standard
- **Gold Standard**: GS4GG requirements
- **Plan Vivo**: Plan Vivo standard
- **DRC REDD**: National REDD framework

### Seed Process

The seed script parses all methodology documents, chunks them at 600 tokens with metadata tags (standard, section, version), embeds using BGE-small, and stores in the methodology vector collection. This runs once (or when standards are updated) with no LLM calls needed — only embedding operations.

---

## 14. Environment Configuration

Key configuration categories include:

- **LLM Providers**: OpenRouter and Cohere API keys, model selection for extraction and chat tasks, fallback model configuration
- **Embedding**: Backend choice (local vs. Cohere), model selection
- **Token Limits**: Maximum tokens for brain JSON, document chunks, methodology chunks, chat history turns, user messages, and output
- **Database**: PostgreSQL connection string with SQLite fallback for development
- **Storage**: Upload directory, ChromaDB directory, maximum upload size, maximum documents per project
- **Auth**: JWT secret, algorithm, and expiration hours
- **App**: Debug mode, environment type, optional Celery broker URL for async processing

---

## 15. Setup & Installation

### Prerequisites

- Python 3.11+
- PostgreSQL 15+ (or SQLite for dev)
- Redis (optional, for Celery async processing)

### Local Dev Setup

The setup process involves cloning the repository, creating a virtual environment, installing dependencies, configuring environment variables from the example file, initializing the database with Alembic migrations, building the methodology knowledge base (one-time operation), and running the development server with hot reload.

### Key Dependencies

The requirements span FastAPI and async database drivers, authentication libraries, document parsing tools (PyMuPDF, python-docx, openpyxl, unstructured, pytesseract), embedding and vector storage (sentence-transformers, ChromaDB, Cohere), agent orchestration (LangGraph, LangChain core), rate limiting (slowapi), optional task queuing (Celery, Redis), and testing tools (pytest, httpx).

---

## 16. Development Roadmap

### Phase 1 — Core Pipeline (Week 1–2)

- Database models and migrations
- Authentication system (register, login, JWT, roles)
- Document upload and parsing (PDF, DOCX, TXT)
- Static chunking and BGE-small embedding
- ChromaDB integration
- Basic brain JSON extraction (fixed schema first)

### Phase 2 — Intelligence Layer (Week 3–4)

- Dynamic schema discovery
- Chunked extraction with schema merging
- Project brain builder
- Methodology KB seeded and queryable
- Chat endpoint with context assembly
- LangGraph agent (basic linear graph)

### Phase 3 — Multi-User + Roles (Week 5)

- Government role and jurisdiction filtering
- Admin endpoints
- Per-user rate limiting
- Project sharing (government viewing regular user projects)

### Phase 4 — Report Generation (Week 6)

- Two-pass report generation (Stage 1–4, Stage 5–8)
- Report persistence and versioning
- Report download (Markdown / PDF export)

### Phase 5 — Production Hardening

- Celery and Redis for async document processing
- PostgreSQL and pgvector migration
- Proper logging and monitoring
- Token usage tracking per user
- Admin dashboard for token/cost visibility

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

Consider graduating to paid models if you want:
- Faster response times (free models have queuing delays under load)
- Better reasoning quality for complex methodology analysis
- Higher RPM limits for concurrent users

Recommended paid alternatives include Claude Haiku 3.5 (cheap) or Gemini Flash 2.0 for chat.

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
