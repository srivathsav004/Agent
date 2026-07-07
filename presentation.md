# Presentation Guide: Agentic AI Demo

> How to walk a tech-savvy audience through this primer in ~30 minutes.

---

## Pre-Demo Setup (5 min before)

1. **Have the README open** on your screen (split view or second monitor)
2. **Prepare a live demo** (even a simple one beats slides):
   - A simple RAG chatbot (e.g., ask questions about a PDF)
   - OR a tool-calling agent (e.g., "What's the weather?" → calls API)
3. **Have these tabs ready**:
   - OpenAI Playground or similar
   - A vector DB dashboard (Qdrant UI / Pinecone)
   - Your code editor with a simple agent script

---

## Opening Hook (2 min)

**Start with a relatable problem:**
> "You know how ChatGPT is great but can't access your company's internal docs, can't run code, and forgets everything after the conversation ends? Agents fix all of that."

**Show, don't tell:**
- Open ChatGPT → ask about your company's internal API → "I don't have access to that"
- Then open your agent demo → ask the same thing → it retrieves from your docs and answers

---

## Section 1: What is an Agent? (3 min)

**Key message:** An agent is an LLM that can use tools and loop until done.

**What to say:**
> "A regular chatbot is like a person who can only talk. An agent is like a person who can talk, Google things, run code, check a database, and keep trying until the job is done."

**Visual:** Point to the simple diagram in the README (Input → LLM → Tools → Output loop).

**Don't over-explain.** If they get it, move on.

---

## Section 2: LLMs & Parameters (3 min)

**Key message:** Bigger isn't always better; understand the trade-offs.

**What to say:**
> "You see models advertised as '7B' or '70B' parameters. Think of it like team size — a 7B model is a smart specialist, 70B is a generalist team. For most tasks, a fine-tuned 7B beats a generic 70B."

**Show the table** from the README. Highlight:
- Small models = edge/fast/cheap
- Large models = complex reasoning
- MoE = efficient scaling

**Skip the math.** Just give intuition.

---

## Section 3: Tokens & Cost (3 min)

**Key message:** Tokens are the currency. Understand how you're charged.

**What to say:**
> "Every word you send and receive costs tokens. Roughly 100 words = 130 tokens. APIs charge per million tokens. The key knobs are temperature — 0 for code, 0.7 for creative writing — and max_tokens to cap output length."

**Show the cost example** from the README.

**Live demo:** Open OpenAI's tokenizer (platform.openai.com/tokenizer) and paste a paragraph. Show how it splits into tokens.

---

## Section 4: RAG — The "Memory" Layer (5 min)

**Key message:** RAG lets LLMs answer questions about data they were never trained on.

**What to say:**
> "LLMs have a knowledge cutoff — they don't know about your company's Q3 report. RAG is the fix: chunk your docs, convert to vectors, store in a vector DB. When someone asks a question, find the most relevant chunks and feed them to the LLM as context."

**Walk through the RAG pipeline diagram** step by step:
1. Load PDFs
2. Chunk into ~500-token pieces
3. Embed each chunk into a vector
4. Store in Qdrant/Pinecone
5. Query → retrieve top 5 chunks → generate answer

**Live demo:**
- Show your vector DB dashboard with some chunks
- Run a query and show the retrieved chunks
- Show how the LLM answers using those chunks

**Emphasize chunking:**
> "Chunk size matters. Too small = loses context. Too big = dilutes relevance. 512 tokens with 20% overlap is a good starting point."

---

## Section 5: Vector Databases (2 min)

**Key message:** It's just a database optimized for "find similar things."

**What to say:**
> "A vector DB is like a search engine for meaning. Instead of keyword matching, it finds 'semantically similar' content using cosine similarity."

**Show the comparison table.** Mention:
- **Qdrant** — open-source, great for self-hosting
- **Pinecone** — managed, fastest to start
- **Chroma** — great for local prototyping

**Don't deep-dive into HNSW or indexing algorithms** unless they ask.

---

## Section 6: Frameworks — LangChain & LangGraph (4 min)

**Key message:** Don't build from scratch; use frameworks that handle the plumbing.

**What to say:**
> "LangChain is the React of AI apps. It gives you chains, agents, memory, and document loaders out of the box. LangGraph adds state machines — so your agent can loop, branch, and handle complex workflows."

**Show the ecosystem diagram** (LangChain → LangGraph → LangSmith → LangServe).

**Mention alternatives briefly:**
> "LlamaIndex is great for heavy RAG. CrewAI for multi-agent teams. Pydantic AI if you want type safety."

**Live demo:** Show a simple LangChain agent script. Highlight:
```python
# The magic is here — the LLM decides which tool to call
agent = Agent(llm="gpt-4o", tools=[search, calculator, email])
response = agent.run("Find sales data and email the report")
```

---

## Section 7: Agentic RAG (3 min)

**Key message:** RAG 1.0 retrieves once. Agentic RAG retrieves, reasons, and retries.

**What to say:**
> "Basic RAG is 'retrieve once, answer once.' But what if the first retrieval sucks? Agentic RAG gives the agent judgment — it can say 'these chunks aren't helpful, let me search differently' or 'I need to look up two things and combine them.'"

**Show the comparison diagram** (Traditional vs. Agentic RAG).

**Mention the patterns:**
- Self-RAG: Agent scores retrieval quality
- Corrective RAG: Falls back to web search if local docs fail
- Multi-hop: Chains retrievals for complex questions

---

## Section 8: Key Concepts & Missing Pieces (3 min)

**Rapid-fire through the 'Key Concepts' section:**

| Concept | One-liner |
|---------|-----------|
| **Fine-tuning vs RAG** | "Fine-tuning teaches behavior; RAG injects facts." |
| **Prompt Engineering** | "Zero-shot = no examples. Few-shot = 2-3 examples. CoT = 'think step by step.'" |
| **Guardrails** | "Safety nets — block bad inputs, fact-check outputs." |
| **MCP** | "USB-C for AI tools — standard way to connect agents to anything." |
| **Multi-Agent** | "Team of specialists instead of one generalist." |

**Ask:** "Any of these you want me to expand on?"

---

## Section 9: Architecture Overview (2 min)

**Key message:** Here's how it all fits together in production.

**What to say:**
> "In production, you have the user interface, APIs, and databases feeding into an agent framework. The framework orchestrates the LLM, vector DB, and memory. Everything is monitored with observability tools like LangSmith."

**Point to the architecture diagram.** Emphasize:
- Vector DB is separate from the LLM
- Memory layer is crucial for continuity
- Observability is non-negotiable in production

---

## Closing & Q&A (2 min)

**Summarize in one sentence:**
> "An agent is an LLM with tools, memory, and the ability to loop until a job is done. RAG gives it knowledge, vector DBs store that knowledge, and frameworks like LangChain make it buildable."

**Leave them with:**
- The README file (they can reference it)
- A GitHub repo or Colab notebook with your demo code
- Offer to do a follow-up deep-dive on any topic

**Expected questions and quick answers:**

| Question | Answer |
|----------|--------|
| "How do I choose a model?" | "Start with GPT-4o or Claude 3.5. Move to open models (Llama 3) when you need self-hosting or cost control." |
| "Is RAG enough or do I need fine-tuning?" | "RAG first, always. Fine-tune only if you need to change behavior/style, not facts." |
| "How do I evaluate if my agent is good?" | "Task success rate, cost per task, and human review of a sample. Use LangSmith to trace every step." |
| "Can this run on-prem?" | "Yes — use Ollama or vLLM for local LLMs, Qdrant or Chroma for local vector DB." |
| "What's the hardest part?" | "Chunking strategy and evaluation. Everyone gets the 'happy path' working; edge cases are the challenge." |

---

## Pro Tips for Delivery

1. **Speak their language** — they're technical, so use terms like "API," "state machine," "pipeline"
2. **Never read slides** — the README is your reference, not a script
3. **Live demos > diagrams** — even a 5-line script running in a notebook is more impressive than a Mermaid chart
4. **It's okay to say "I don't know"** — but follow with "...but here's how I'd find out"
5. **End 2 minutes early** — gives buffer for questions and makes you look prepared

---

## Emergency Backup Plan

If the live demo breaks:
1. **Have screenshots ready** of the demo working
2. **Switch to the architecture diagram** — walk through the flow verbally
3. **Use the tokenizer demo** — it's hard to break and always works

---

*Good luck. You've got this.* 🎯




# How to Present This — Speaker Guide

Goal: ~15-20 min walkthrough for a tech person. Don't read the README aloud — talk over it, use it as your slide/scroll reference.

---

## Flow & Timing

| # | Section | Time | Say this, roughly |
|---|---|---|---|
| 1 | Tokens & Params | 3 min | "Before agents, quick foundation. LLMs read tokens, not words — that's why cost is measured per token. Params are basically how much 'knowledge capacity' the model has — 7B vs 70B vs frontier models like GPT-4/Claude class." |
| 2 | What's an agent | 4 min | "Normally an LLM just answers from what it was trained on. An agent can *act* — call a tool, check a result, decide the next step. It's a loop, not a single call." → show the ReAct diagram. |
| 3 | LangChain vs LangGraph | 3 min | "LangChain is the toolbox — prompts, tool wrappers. LangGraph is for when your agent's logic has loops/branches — you model it as a graph instead of a straight pipeline." → show the table, then the graph diagram. |
| 4 | RAG | 4 min | "LLMs don't know your private docs or anything recent. RAG = fetch relevant info first, then answer. Chunk the doc, embed each chunk into a vector, store in a vector DB like Qdrant, and at query time find the closest matches." → walk the RAG diagram left to right. |
| 5 | Agentic RAG | 3 min | "Normal RAG retrieves once and answers. Agentic RAG lets the agent *decide* — retrieve again, try a different source, skip retrieval if not needed." → show that diagram briefly, don't over-explain, it's just RAG + the agent loop combined. |
| 6 | Wrap-up / what's next | 2 min | Hit the "what's missing" bullets fast — fine-tuning, tool calling, multi-agent, memory, guardrails, MCP. One sentence each, don't dive deep unless he asks. |

---

## Presentation Tips

- **Lead with the "why", not the "what."** E.g. "Normal chatbot vs agent" — show the *gap* first, then explain how it's filled.
- **Use the diagrams as anchors** — point at boxes as you talk, don't just read them.
- **If he asks something you don't know**: it's fine to say "good question, let me note that and check" — better than guessing on a tech audience.
- **Have one concrete example ready** for each concept if he wants it made real, e.g.:
  - Agent: "an agent that checks weather API then books a slot only if it's sunny"
  - RAG: "answering questions about your company's internal PDFs that GPT was never trained on"
  - Agentic RAG: "an agent that searches your docs, and if it doesn't find enough, also searches the web"
- **Keep the demo, if you have one, AFTER the concepts** — concepts first builds the vocabulary so the demo actually lands.
- **Don't over-index on Qdrant specifically** — mention it's *a* vector DB (like Pinecone/Weaviate/FAISS), not the only option, unless your demo specifically uses it.

---

## Anticipated Questions (and quick answers)

- **"Is this basically just a chatbot with function calling?"** → Yes, largely — the "agent" behavior emerges from tool calling + a loop that decides when to keep calling tools.
- **"Why not just increase context window instead of RAG?"** → You can fit more, but it's slower/costlier per call, and doesn't scale to huge document sets. RAG retrieves only what's relevant.
- **"How is chunk size decided?"** → Trade-off: smaller chunks = more precise retrieval but less context per chunk; usually tuned empirically (e.g. 500-1000 tokens with overlap).
- **"What stops the agent from looping forever?"** → Usually a max iteration count or a clear "done" condition set in the framework (e.g. LangGraph lets you define exit edges).

---

## One-Line Closer

> "So really — LLM is the brain, tools give it hands, RAG gives it a memory of your specific data, and LangGraph is just how we wire the decision-making so it doesn't run wild."
