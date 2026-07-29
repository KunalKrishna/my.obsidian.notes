A few high-signal resources per phase — a mix of **free** and one or two **paid** picks. I've leaned toward things that were current in mid-2026, since this stack goes stale fast.

**Note on links:** I've given official/home pages and the resource's exact name + author so you can find it even if a deep URL shifts. In this field, **prefer official docs over any tutorial** for API surface, and check a course's "last updated" date before investing time — anything not mentioning agents, MCP, or modern RAG in 2026 is dated.

**Two anchor resources for the whole month (buy/bookmark these):**
- 📖 **Chip Huyen — _AI Engineering_** (O'Reilly, 2025). The definitive framing for this exact role; the roadmap's structure follows it. _Paid (book)._
- 🎓 **LangChain Academy** — `academy.langchain.com`. Free, official, and the most current source for LangChain/LangGraph + LangSmith evaluation.

---
## Phase 0 — Setup & Framing

- **Chip Huyen — _AI Engineering_, Ch. 1–2** (the 3-layer stack; what AI engineering is). _Paid._
- **Andrej Karpathy — "Deep Dive into LLMs like ChatGPT"** (YouTube). Best refresher on inference-time behavior; you have the training theory already. _Free._
- Roadmap overviews to skim: **Dataquest "AI Engineer Roadmap"** (`dataquest.io/blog/ai-engineer-roadmap`) and **KDnuggets "The Roadmap to Becoming an LLM Engineer in 2026."** _Free._
## Phase 1 — LLM API Fluency & Prompt Engineering

- **Anthropic docs — Prompt Engineering guide** (`docs.anthropic.com`) and the **Anthropic "Prompt Engineering Interactive Tutorial"** (GitHub). The best hands-on prompting resource. _Free._
- **OpenAI Platform docs** (`platform.openai.com/docs`) — tool calling, structured outputs, streaming. _Free._
- **DeepLearning.AI — "ChatGPT Prompt Engineering for Developers"** (Andrew Ng + Isa Fulford). Short, foundational. _Free._
- **Pydantic docs** + the **`instructor`** library docs — for reliable structured outputs. _Free._

## Phase 2 — Evaluation Foundations ★ your gap

- **Hamel Husain — "Your AI Product Needs Evals"** (blog, `hamel.dev`). The single best essay on why/how to build evals. _Free — read this first._
- **DeepEval docs** (`deepeval.com`) — code-first, pytest-style CI evals; skim the metrics catalog and LLM-as-judge guide. _Free._
- **RAGAS docs** (`docs.ragas.io`) — RAG metric definitions (faithfulness, answer relevancy, context precision/recall). _Free._
- **DeepLearning.AI — "Evaluating AI Agents"** and **"Building and Evaluating Advanced RAG"** (Jerry Liu + Anupam Datta). _Free short courses._
- **Langfuse docs** (`langfuse.com/docs`) — tracing + online evals. _Free / OSS._

## Phase 3 — RAG

- **freeCodeCamp — "RAG From Scratch"** (long-form YouTube, taught by a LangChain engineer). Best free deep walkthrough. _Free._
- **DeepLearning.AI — "Retrieval Augmented Generation (RAG)"** (Zain Hasan, 5 modules) — strongest credentialed starting point. _Free._
- **LangChain RAG tutorial** (`python.langchain.com` → Tutorials → RAG). _Free._
- **Weights & Biases — "RAG++: From POC to Production."** Excellent once you know the basics. _Free._
- **Pinecone Learn — RAG / vector-search handbook** (`pinecone.io/learn`). Great on retrieval internals. _Free._
- Evaluation-specific: reuse **RAGAS docs** from Phase 2 for the eval report.

## Phase 4 — Agents & Tool Use (LangGraph)

- **LangChain Academy — "Introduction to LangGraph"** (official, free). Do this one properly. _Free._
- **Anthropic — "Building Effective Agents"** (engineering blog). The clearest thinking on when _not_ to over-engineer an agent. _Free._
- **DeepLearning.AI — "AI Agents in LangGraph"** (Harrison Chase + Tavily) and **Andrew Ng — "Agentic AI"** (the 4 design patterns: reflection, tool use, planning, multi-agent — these transfer across every framework). _Free._
- **Hugging Face — AI Agents Course** (free certificate; covers LangGraph, smolagents, observability). _Free._
- _Alternatives to be aware of:_ CrewAI docs, PydanticAI docs, Microsoft AutoGen docs.

## Phase 5 — MCP (Model Context Protocol)

- **Official MCP docs** (`modelcontextprotocol.io`) + the **specification** and **FastMCP** docs/GitHub. Primary source; the protocol is the durable bet. _Free._
- **Anthropic — "Introduction to MCP"** (`anthropic.com/news/model-context-protocol`) and the **DeepLearning.AI — "MCP: Build Rich-Context AI Apps with Anthropic"** short course. _Free._
- Skim a current state-of-the-ecosystem piece (server registry, security like tool poisoning) — the _DEV.to_ "Complete Guide to MCP in 2026" is a solid one. _Free._

## Phase 6 — Fine-tuning (LoRA / QLoRA)

- **Hugging Face — LLM Course** (`huggingface.co/learn`) fine-tuning chapters + **PEFT** and **TRL** docs. The de-facto reference, written by the library authors. _Free._
- **Unsloth — notebooks & docs** (`docs.unsloth.ai`, Colab notebooks). Fastest path to a working LoRA/QLoRA run on free/cheap GPU. _Free / OSS._
- **Sebastian Raschka — LoRA & QLoRA articles** (`sebastianraschka.com`). Best conceptual explanations of _why_ it works. _Free._
- **DeepLearning.AI — "Finetuning Large Language Models"** (Sharon Zhou). Good short primer on the _when/why_. _Free._

## Phase 7 — Productionization / LLMOps & Guardrails

- **Full Stack Deep Learning — LLM Bootcamp** (`fullstackdeeplearning.com`). Free, production-focused lectures on serving & LLMOps. _Free._
- **Made With ML** (Goku Mohandas, `madewithml.com`). The clearest free MLOps foundation. _Free._
- **FastAPI docs** + **vLLM docs** (serving) + **Langfuse docs** (observability/cost). _Free._
- **Guardrails-AI** and **NVIDIA NeMo Guardrails** docs — I/O validation, prompt-injection defense. _Free / OSS._

## Phase 8 — Capstone & Portfolio

- Reuse everything above. For packaging: study a few **top AI-engineering GitHub READMEs** (architecture diagram + eval table + demo GIF) and copy the structure.
- Diagrams: **Excalidraw** or **Mermaid** for architecture; **Loom** for a 90-second demo.

## Interview Prep (threaded)

- 📖 **Chip Huyen — _Designing Machine Learning Systems_** and her free **"Machine Learning Interviews" GitHub book** (`huyenchip.com/ml-interviews-book`). _Book paid; interview book free._
- **Evidently AI blog** and **"ML System Design" write-ups** for LLM-app design questions (RAG design, agent design, "how would you evaluate this?"). _Free._
- Keep DSA warm on **LeetCode** — light maintenance only. _Free._

---

### Structured paid paths (optional — only if you prefer guided courses over self-assembly)

These bundle much of the roadmap into one place; pick **one at most**, don't collect certificates:

- **Ed Donner — "Master LLM Engineering & AI Agents"** (Udemy). ~14 projects across OpenAI/Claude/HF/Ollama, RAG with LangChain+Chroma, LoRA/QLoRA, agents (LangGraph, OpenAI Agents SDK, AutoGen, **MCP**). Explicitly built for interview-ready project code. _Paid (cheap on sale)._
- **IBM — "RAG and Agentic AI Professional Certificate"** (Coursera). Advanced, credentialed; strong **MCP + LangGraph + CrewAI** coverage; good if your employer reimburses Coursera. _Paid._
- **DataCamp — "Associate AI Engineer for Developers"** track. ~80 hrs, hiring-data-driven, interactive coding. _Paid (subscription)._


----

You need a working-level understanding of four concepts: 
1. **tokens** (the units models actually process), 
2. **embeddings** (how tokens become vectors in high-dimensional space), 
3. **attention** (how the model weighs relationships between tokens), and 
4. **the transformer** block as the repeating architectural unit.