# Agentic AI: A Primer for Techies

> A concise guide to understanding and building AI agents — from first principles to production.

---

## Table of Contents
1. [What is an AI Agent?](#1-what-is-an-ai-agent)
2. [LLMs: The Brain](#2-llms-the-brain)
3. [Tokens & Cost](#3-tokens--cost)
4. [RAG: Giving LLMs Memory](#4-rag-giving-llms-memory)
5. [Vector Databases](#5-vector-databases)
6. [Agentic Frameworks](#6-agentic-frameworks)
7. [Building an Agent](#7-building-an-agent)
8. [Agentic RAG](#8-agentic-rag)
9. [Key Concepts You Might Be Missing](#9-key-concepts-you-might-be-missing)
10. [Architecture Overview](#10-architecture-overview)

---

## 1. What is an AI Agent?

An **AI Agent** is an LLM-powered system that:
- **Perceives** its environment (reads data, receives prompts)
- **Reasons** about what to do next
- **Acts** by calling tools (APIs, databases, code execution)
- **Loops** until the goal is achieved

```
┌─────────────┐
│   Input     │
└──────┬──────┘
       ▼
┌─────────────┐     ┌─────────────┐
│    LLM      │────▶│   Tools     │
│  (Brain)    │◀────│  (APIs, DB) │
└──────┬──────┘     └─────────────┘
       │
       ▼
┌─────────────┐
│   Output    │
└─────────────┘
```

**vs. Traditional Chatbot:**
| Feature | Chatbot | Agent |
|---------|---------|-------|
| Memory | Session-only | Persistent + Retrieval |
| Tools | None | APIs, code, databases |
| Reasoning | Single-turn | Multi-step planning |
| Autonomy | Reactive | Proactive |

---

## 2. LLMs: The Brain

### What are "Billion Parameters"?

| Model Size | Parameters | Analogy | Best For |
|------------|------------|---------|----------|
| Small | 1B–7B | Smart intern | Edge devices, simple tasks |
| Medium | 13B–30B | Senior dev | Balanced speed & quality |
| Large | 70B+ | Principal engineer | Complex reasoning, coding |
| MoE (Mixture of Experts) | 8x7B, 8x22B | Team of specialists | Efficient large-scale inference |

**Key insight:** More parameters ≠ always better. A 7B model with fine-tuning often beats a 70B model for domain-specific tasks.

### Popular Models (2025)
- **GPT-4o / o3** (OpenAI) — General purpose, multimodal
- **Claude 3.5 Sonnet** (Anthropic) — Long context, reasoning
- **Llama 3.1/3.2** (Meta) — Open weights, self-hostable
- **DeepSeek-V3** — Open, MoE architecture
- **Gemini 2.5 Pro** (Google) — Multimodal, long context

---

## 3. Tokens & Cost

### What is a Token?
A token is a sub-word unit. Roughly:
- **1 token ≈ 0.75 words** (English)
- **1 word ≈ 1.3 tokens**

| Text | Tokens |
|------|--------|
| "Hello world" | 3 |
| This paragraph (~50 words) | ~65 |
| A 500-word blog post | ~650 |

### Token Utilization Parameters
When calling an LLM API, you specify:

| Parameter | What It Does | Typical Value |
|-----------|--------------|---------------|
| `max_tokens` | Hard limit on output length | 256–4096 |
| `temperature` | Creativity vs. determinism (0=robotic, 2=chaotic) | 0.0–1.0 |
| `top_p` | Nucleus sampling (alternative to temperature) | 0.1–1.0 |
| `frequency_penalty` | Reduces repetition | 0.0–2.0 |
| `presence_penalty` | Encourages new topics | 0.0–2.0 |

### Cost Calculation
```
Total Cost = (Input Tokens × Input Price) + (Output Tokens × Output Price)

Example: GPT-4o
- Input: $2.50 / 1M tokens
- Output: $10.00 / 1M tokens
- 1M input + 200K output = $2.50 + $2.00 = $4.50
```

---

## 4. RAG: Giving LLMs Memory

**RAG = Retrieval-Augmented Generation**

LLMs have a **knowledge cutoff** and **no access to private data**. RAG fixes both.

### How RAG Works

```
┌─────────────────────────────────────────┐
│           RAG Pipeline                  │
├─────────────────────────────────────────┤
│                                         │
│  1. LOAD        2. CHUNK      3. EMBED  │
│  ┌─────┐       ┌──────┐      ┌──────┐   │
│  │PDFs │──────▶│Chunks│─────▶│Vectors│  │
│  │Docs │       │(512tk)│      │(768d) │  │
│  │APIs │       └──────┘      └──────┘   │
│  └─────┘           │            │       │
│                    ▼            ▼       │
│              ┌─────────┐  ┌──────────┐  │
│              │Overlap  │  │ Vector DB│  │
│              │(20%)    │  │(Qdrant)  │  │
│              └─────────┘  └──────────┘  │
│                                         │
│  4. QUERY ───────────────────────────▶  │
│     "How do I deploy?"                  │
│         │                               │
│         ▼                               │
│  5. RETRIEVE (Top-K=5)                  │
│         │                               │
│         ▼                               │
│  6. AUGMENT PROMPT                      │
│     [Context] + [User Query]            │
│         │                               │
│         ▼                               │
│  7. GENERATE                            │
│     LLM answers with retrieved facts    │
│                                         │
└─────────────────────────────────────────┘
```

### Chunking Strategies

| Strategy | Description | Best For |
|----------|-------------|----------|
| **Fixed-size** | Split every N tokens | General docs |
| **Recursive** | Split by headers, then by size | Structured docs |
| **Semantic** | Split at semantic boundaries | Narrative text |
| **Agentic** | LLM decides chunk boundaries | Complex documents |

**Chunk Size Rule of Thumb:** 256–1024 tokens with 10–20% overlap.

### Embeddings

Embeddings convert text into **dense vectors** (arrays of numbers) where similar meanings are close together in vector space.

```
"cat"    → [0.12, -0.45, 0.89, ...]  (768 or 1536 dimensions)
"kitten" → [0.15, -0.42, 0.85, ...]  ← close to "cat"
"car"    → [-0.33, 0.71, -0.12, ...] ← far from "cat"
```

Popular embedding models:
- **OpenAI `text-embedding-3-large`** — 3072 dims, best quality
- **Cohere `embed-english-v3`** — 1024 dims, fast
- **BGE-M3** (BAAI) — Open, multilingual
- **Nomic Embed** — Open, 768 dims

---

## 5. Vector Databases

A **Vector DB** stores embeddings and performs **similarity search** (find vectors closest to a query).

```
┌─────────────────────────────────────┐
│        Vector Database              │
├─────────────────────────────────────┤
│                                     │
│  Collection: "company_docs"         │
│  ┌─────────┬────────────────────┐   │
│  │ ID      │ Vector (768 dims)  │   │
│  ├─────────┼────────────────────┤   │
│  │ chunk_1 │ [0.1, -0.2, ...]   │   │
│  │ chunk_2 │ [0.3, 0.1, ...]    │   │
│  │ chunk_3 │ [-0.1, 0.4, ...]   │   │
│  └─────────┴────────────────────┘   │
│                                     │
│  Query: "deployment guide"          │
│  → Returns: chunk_1, chunk_3 (Top-K)│
│                                     │
│  Similarity Metrics:                │
│  • Cosine (most common)             │
│  • Euclidean                        │
│  • Dot Product                      │
│                                     │
└─────────────────────────────────────┘
```

### Popular Vector DBs

| Database | Type | Best For |
|----------|------|----------|
| **Qdrant** | Open-source, Rust | Self-hosted, hybrid search |
| **Pinecone** | Managed SaaS | Zero-ops, fast start |
| **Weaviate** | Open-source | Graph + vector hybrid |
| **Chroma** | Embedded | Prototyping, local dev |
| **Milvus/Zilliz** | Distributed | Enterprise scale |
| **pgvector** | Postgres extension | Existing PG users |

---

## 6. Agentic Frameworks

### LangChain

**LangChain** is the most popular framework for building LLM apps. It provides:
- **Chains**: Sequences of calls (LLM → Tool → LLM)
- **Agents**: LLM decides which tool to use next
- **Memory**: Conversation persistence
- **Document Loaders**: PDF, web, database ingestion

```
LangChain Ecosystem:
┌─────────────┐
│  LangChain  │  ← Core framework (Python/JS)
├─────────────┤
│  LangGraph  │  ← State machines for multi-agent flows
├─────────────┤
│ LangSmith   │  ← Observability & tracing
├─────────────┤
│ LangServe   │  ← Deploy chains as REST APIs
└─────────────┘
```

### LangGraph

**LangGraph** adds **cycles** to LangChain — essential for agents that need to loop (think → act → observe → repeat).

```
LangGraph State Machine:

┌─────────┐     ┌─────────┐     ┌─────────┐
│  START  │────▶│  Agent  │────▶│  Tool   │
└─────────┘     │  (LLM)  │◀────│ (Action)│
                └────┬────┘     └─────────┘
                     │
                     ▼
                ┌─────────┐
                │  END    │
                └─────────┘

The agent loops between reasoning and acting until done.
```

### Other Frameworks

| Framework | Focus | Best For |
|-----------|-------|----------|
| **LlamaIndex** | Data ingestion & RAG | Complex document pipelines |
| **CrewAI** | Multi-agent orchestration | Team-of-agents workflows |
| **AutoGen** (Microsoft) | Conversational agents | Multi-agent chat |
| **Pydantic AI** | Type-safe agents | Production Python apps |
| **Semantic Kernel** (Microsoft) | Enterprise integration | .NET/Python enterprise |

---

## 7. Building an Agent

### Minimal Agent (Pseudocode)

```python
# 1. Define Tools
tools = [search_web, query_database, send_email]

# 2. Create Agent
agent = Agent(
    llm="gpt-4o",
    tools=tools,
    system_prompt="You are a helpful assistant. Use tools when needed."
)

# 3. Run
response = agent.run("Find the latest sales report and email it to the team")

# Behind the scenes:
# Step 1: LLM decides → call query_database
# Step 2: LLM receives result → decides → call send_email
# Step 3: LLM confirms → return success message
```

### Agent Components

```
┌─────────────────────────────────────────┐
│           Agent Architecture            │
├─────────────────────────────────────────┤
│                                         │
│  ┌─────────────┐                        │
│  │   Planner   │  ← Decides strategy    │
│  │  (LLM/CoT)  │                        │
│  └──────┬──────┘                        │
│         │                               │
│         ▼                               │
│  ┌─────────────┐                        │
│  │   Executor  │  ← Calls tools         │
│  │  (Tool Use) │                        │
│  └──────┬──────┘                        │
│         │                               │
│         ▼                               │
│  ┌─────────────┐                        │
│  │   Memory    │  ← Stores context      │
│  │ (Short/Long)│                        │
│  └──────┬──────┘                        │
│         │                               │
│         ▼                               │
│  ┌─────────────┐                        │
│  │  Observer   │  ← Evaluates results   │
│  │ (Self-check)│                        │
│  └─────────────┘                        │
│                                         │
└─────────────────────────────────────────┘
```

---

## 8. Agentic RAG

**Agentic RAG** = RAG + Agent reasoning. The agent doesn't just retrieve once — it **iteratively searches, reasons, and refines**.

### Traditional RAG vs. Agentic RAG

```
Traditional RAG:                    Agentic RAG:
┌──────────┐                       ┌──────────┐
│  Query   │                       │  Query   │
└────┬─────┘                       └────┬─────┘
     │                                │
     ▼                                ▼
┌──────────┐                       ┌──────────┐
│ Retrieve │                       │  Plan    │
│  (1x)    │                       │(Strategy)│
└────┬─────┘                       └────┬─────┘
     │                                │
     ▼                                ▼
┌──────────┐                       ┌──────────┐
│ Generate │                       │ Retrieve │
│  (1x)    │                       │ (Maybe   │
└──────────┘                       │ multiple)│
                                   └────┬─────┘
                                       │
                                       ▼
                                  ┌──────────┐
                                  │ Evaluate │
                                  │ (Good    │
                                  │ enough?) │
                                  └────┬─────┘
                                       │
                              No ◀─────┘─────▶ Yes
                              │                 │
                              ▼                 ▼
                         ┌──────────┐      ┌──────────┐
                         │ Refine   │      │ Generate │
                         │ Query    │      │  Answer  │
                         └──────────┘      └──────────┘
```

### Agentic RAG Patterns

| Pattern | How It Works |
|---------|--------------|
| **Self-RAG** | Agent evaluates if retrieved chunks are useful; discards bad ones |
| **Corrective RAG** | Agent routes to web search if local retrieval is insufficient |
| **Adaptive RAG** | Agent chooses retrieval strategy based on query complexity |
| **Multi-Hop RAG** | Agent chains multiple retrievals to answer complex questions |

---

## 9. Key Concepts You Might Be Missing

### Prompt Engineering
- **Zero-shot**: No examples, just instructions
- **Few-shot**: 2–5 examples in the prompt
- **Chain-of-Thought (CoT)**: "Let's think step by step..."
- **ReAct**: Reasoning + Acting pattern (the foundation of agents)

### Fine-Tuning vs. RAG

| | Fine-Tuning | RAG |
|---|-------------|-----|
| **What** | Retrain model weights | Inject context at inference |
| **Cost** | High (GPU, data prep) | Low (vector search) |
| **Updates** | Retrain needed | Just add documents |
| **Best For** | Behavior/style changes | Factual knowledge |

### Evaluation Metrics
- **Retrieval**: Hit Rate, MRR, NDCG
- **Generation**: BLEU, ROUGE, Faithfulness, Answer Relevance
- **Agent**: Task Success Rate, Steps to Complete, Cost per Task

### Guardrails
- **Input**: Block harmful prompts, PII detection
- **Output**: Fact-checking, toxicity filtering, compliance
- **Tools**: Rate limiting, sandboxed execution

### MCP (Model Context Protocol)
A standard for connecting LLMs to external data sources and tools. Think "USB-C for AI agents."

### Multi-Agent Systems
Multiple specialized agents collaborating:
- **Planner Agent**: Breaks down tasks
- **Coder Agent**: Writes code
- **Reviewer Agent**: Checks output
- **Orchestrator**: Manages the team

---

## 10. Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    Production Agent Stack                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐        │
│  │   User      │   │   APIs      │   │  Databases  │        │
│  │  Interface  │   │  (REST/gRPC)│   │  (SQL/NoSQL)│        │
│  └──────┬──────┘   └──────┬──────┘   └──────┬──────┘        │
│         │                 │                 │               │
│         └─────────────────┼─────────────────┘               │
│                           ▼                                 │
│                  ┌─────────────────┐                        │
│                  │ Agent Framework │                        │
│                  │ (LangChain/Graph)│                       │
│                  └────────┬────────┘                        │
│                           │                                 │
│         ┌─────────────────┼─────────────────┐               │
│         ▼                 ▼                 ▼               │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │    LLM      │  │   Vector    │  │   Memory    │          │
│  │ (GPT/Claude)│  │    DB       │  │  (Redis/DB)│           │
│  │             │  │ (Qdrant/etc)│  │             │          │
│  └─────────────┘  └─────────────┘  └─────────────┘          │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              Observability (LangSmith/Phoenix)      │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Quick Reference Cheat Sheet

| Term | One-Liner |
|------|-----------|
| **Agent** | LLM + Tools + Memory + Loop |
| **RAG** | Retrieve docs → Inject into prompt → Generate |
| **Vector DB** | Database for similarity search on embeddings |
| **Embedding** | Text → Numbers (vector) |
| **Chunking** | Splitting docs into bite-sized pieces |
| **Token** | Sub-word unit (~0.75 words) |
| **Temperature** | Creativity dial (0 = deterministic) |
| **LangChain** | Framework for LLM apps |
| **LangGraph** | Cycles/state machines for agents |
| **Fine-tuning** | Retraining model on custom data |
| **Guardrails** | Safety checks on input/output |
| **MCP** | Standard protocol for AI tools |

---

## Further Reading
- [LangChain Docs](https://python.langchain.com/)
- [LangGraph Docs](https://langchain-ai.github.io/langgraph/)
- [Qdrant Docs](https://qdrant.tech/documentation/)
- [LlamaIndex Docs](https://docs.llamaindex.ai/)
- [Building LLM Apps by Simon Willison](https://simonwillison.net/)

---

*Built for a 30-minute demo. Revise, present, impress.* 🚀
