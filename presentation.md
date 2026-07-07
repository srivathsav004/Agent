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
