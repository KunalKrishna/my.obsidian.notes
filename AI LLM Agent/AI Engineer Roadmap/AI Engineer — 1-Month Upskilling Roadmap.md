**Built for:** an experienced software engineer (Java/SpringBoot, Python basics, strong DSA) with foundational LLM theory (tokenization, transformers, backprop) but **no applied AI projects and no evaluation know-how**.

**Your settings:** ~3–4 hrs/day (≈100 hrs over the month) · **application layer + a fine-tuning module** · goal = **job-ready portfolio + interviews**.

**Why this plan looks the way it does:** you already have the two things most people spend months on — software engineering fundamentals and the theory of how LLMs work. So we _skip_ "learn Python", "learn how transformers work", "learn system design basics" and spend your hours almost entirely on the **gap**: building real AI systems and — above all — **evaluating** them. Evaluation is your stated weak spot and it's also the single biggest differentiator in AI-engineer hiring in 2026, so it is treated as a first-class skill and threaded through every phase, not bolted on at the end.

---
## The one guiding principle
> An AI engineer is not someone who can call an LLM. It's someone who can **tell whether the output is right, know _why_ it's wrong, and drive the error down to a tolerable level** — with numbers, not vibes. Everything below builds toward that.
---
## Pick ONE stack and commit (alternatives in brackets — know they exist, don't chase them) 

| Layer             | Primary choice (learn this)                                                         | Alternatives (awareness only)                                                         |
| ----------------- | ----------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| Language          | **Python** (async, `pydantic`, typing)                                              | —                                                                                     |
| LLM API           | **Anthropic Claude** + **OpenAI** (master one, know the other)                      | Google Gemini; **Ollama** for free local experimentation                              |
| Orchestration     | **LangChain** (building blocks) + **LangGraph** (stateful agents)                   | LlamaIndex (RAG-heavy), PydanticAI (type-safe), **CrewAI** (multi-agent), **AutoGen** |
| Vector DB         | **Chroma** (dev) → **pgvector / Qdrant** (prod)                                     | Pinecone (managed), Weaviate, Milvus, FAISS                                           |
| Retrieval quality | Cohere / OpenAI embeddings + **cross-encoder reranker**                             | Voyage, BGE, Jina                                                                     |
| **Evaluation**    | **RAGAS** (RAG) + **DeepEval** (code/CI evals) + **Langfuse** (tracing)             | LangSmith, Braintrust, Arize Phoenix, Promptfoo                                       |
| Fine-tuning       | **HuggingFace `transformers` + `peft` + `trl`** (LoRA/QLoRA); **Unsloth** for speed | Axolotl; MLX (Apple Silicon)                                                          |
| Serving/deploy    | **FastAPI + Docker**; vLLM for open models                                          | Modal, Runpod, AWS Bedrock, cloud endpoints                                           |
| Guardrails        | Pydantic validation; Guardrails-AI / NeMo Guardrails                                | —                                                                                     |
**Rule of thumb the field agrees on:** the LLM part is ~20% of the job; the other 80% is engineering you already know. Keep the architecture as simple as the problem allows.

---
## How to work through this

1. **Build in public.** Every phase ends in a GitHub push with a real README. This _is_ your portfolio.
2. **POC before deep-dive** on the heavy topics (RAG, Agents, Fine-tuning): get the naive version working end-to-end first, _then_ learn what makes it good.
3. **Evals travel with every build.** No project is "done" until it has a small eval set and a metric.
4. **One variable at a time.** When quality is bad, change one thing, re-measure. This is the core loop.
---
# The Roadmap (do these in order)

## Phase 0 — Setup & Framing · Days 1–2 (~5 hrs)

Get oriented and remove all friction so you never fight your tools again.
- Internalize the **3-layer AI stack**: 
	- _application_ (prompts, context, evaluation, the product — where you'll live) / 
	- _model_ (training, fine-tuning) / 
	- _infrastructure_ (serving, compute). 
	- AI engineering = building on foundation models that already exist.
- Set up: Python env (`uv` or `poetry`), API keys in `.env`, a cost/usage dashboard, a reusable repo template, notebooks.
- **Light** Python-for-AI refresher (you already code): `async`/`await`, `pydantic` models, type hints, `httpx`, streaming responses.
- Refresh the _inference-time_ concepts you didn't cover in coursework (you know training theory): tokens & context windows, temperature/top-p sampling, embeddings vs generation, the cost/latency triangle.
- **Deliverable:** a `hello_llm.py` that calls the API, streams the response, handles retries/rate-limits, and **logs tokens + cost per call**.

## Phase 1 — LLM API Fluency & Prompt Engineering · Days 3–5 (~10 hrs)

Turn the model into a reliable component.
- Go deep on **one provider's API**: messages, system prompts, streaming, **tool/function calling**, **structured outputs** (JSON mode / response schemas), stop reasons, token accounting, retry/fallback logic.
- **Prompt engineering as engineering**: system-prompt design, few-shot example placement, chain-of-thought, explicit output contracts, prompt versioning.
- **Structured outputs with Pydantic** — the backbone of everything downstream (parse, validate, fail loudly).
- **First taste of evals:** how do you _know_ a prompt is good? Build a tiny eval — a handful of test inputs + expected properties + a basic **LLM-as-judge** score. Meet the idea of a **golden dataset**.
- **Mini-project:** a structured-extraction tool (e.g., résumé → JSON, or invoice → fields) **with a small eval set scoring accuracy.**

## Phase 2 — Evaluation Foundations · Days 6–7 (~7 hrs) ★ _your gap — placed early on purpose_

This phase directly answers your question: _"How do I know the output is correct, and how do I get it within tolerable error?"_
- **The eval taxonomy** (three families):
    - _Deterministic / reference-based_ — exact match, F1, BLEU/ROUGE — and **why these are weak for open-ended generation**.
    - _Semantic_ — embedding-similarity to reference answers.
    - _Rubric-based / LLM-as-judge_, and _human eval_ with clear rubrics.
- **Metrics that matter, by task:** RAG → _faithfulness, answer relevancy, context precision/recall_; agents → _task success, tool-call correctness, trajectory_; chat → _helpfulness, coherence, safety_.
- **Build a golden dataset**; run **regression tests**; practice **eval-driven development** (write the eval with the feature).
- **LLM-as-judge, done right:** writing judge prompts, pointwise vs pairwise, calibration and bias pitfalls, and its blind spot (it can't reliably catch _fluent-but-factually-wrong_).
- **The correction loop** = _measure → diagnose (retrieval? prompt? model? data?) → change ONE thing → re-measure_. Setting thresholds and pass/fail **gates**.
- Tools: **DeepEval** (pytest-style CI evals), **RAGAS** (RAG), **Langfuse** (tracing). _[LangSmith / Braintrust / Phoenix / Promptfoo = alternatives.]_
- **Deliverable:** wrap Phase 1's extraction tool in a **DeepEval test suite with pass/fail thresholds running in GitHub Actions**. Now you have CI for AI quality — a strong interview signal.

## Phase 3 — Retrieval-Augmented Generation (RAG) · Days 8–13 (~20 hrs) ★ _heavy — POC first_

### ▶ POC first (Day 8, ~3 hrs)

Build the **naive pipeline** end-to-end over your _own_ documents, no magic: load → chunk → embed → store in Chroma → retrieve top-k → stuff into context → answer. Understand every step before improving any of them.

### Then go deep

- **Chunking strategies** (fixed, recursive, semantic, structure-aware) — chunking quietly dominates RAG quality.
- **Embeddings**: model choice, dimensions, cost, query-vs-document asymmetry.
- **Vector DBs**: Chroma (dev) → pgvector/Qdrant (prod); **metadata filtering**.
- **Hybrid search** (dense + sparse/BM25) and **reranking** (cross-encoder / Cohere) to lift precision.
- **Query transformations**: multi-query, HyDE, decomposition, **semantic routing** across sources.
- **Context assembly**: ordering, dedup, "lost in the middle," **citations**.
- _Awareness only:_ agentic / self-correcting RAG, GraphRAG.

### RAG evaluation (~4 hrs, non-negotiable)

**RAGAS** metrics (faithfulness, answer relevancy, context precision/recall); build a RAG eval set; **diagnose whether a failure is retrieval or generation** — this is exactly the "is it correct and how do I fix it" skill, applied to RAG.

### ▶ PROJECT #1 (portfolio) — "Chat with your documents"

Over a real corpus you care about, with **hybrid search + reranking + citations**, a **RAGAS eval report in the README showing before/after metrics**, deployed via FastAPI + a minimal UI (Streamlit/Gradio).

## Phase 4 — Agents & Tool Use · Days 14–19 (~20 hrs) ★ _heavy — POC first_

### Foundations

- **Tool/function calling** (you tasted it in Phase 1): model returns a structured call → you execute → feed the result back. The **ReAct** pattern (reason + act).
- **The autonomy spectrum**: deterministic workflow → constrained agentic → autonomous agent. Agents are _less_ predictable but handle open-ended tasks. Prefer the least autonomy that solves the problem.

### ▶ POC first (~3 hrs)

A single-tool agent (calculator + web search) with **raw tool calling, no framework** → then a 2–3 tool ReAct agent. Feel the mechanics before the abstraction.

### Then go deep

- **LangGraph** (primary): stateful graphs, nodes/edges, conditional routing, cycles, **checkpointing/persistence**, **human-in-the-loop**, multi-agent patterns (supervisor). _[CrewAI = role-based business workflows; PydanticAI = type-safe; AutoGen = alt. Learn LangGraph; know these names.]_
- **Memory**: short-term (conversation) vs long-term; summarization; state management.
- **Async patterns** (your SpringBoot background shines here): LLM calls are slow — use queue/worker patterns so agent runs don't time out web requests.

### Agent evaluation

Task success rate, **tool-call correctness**, **trajectory/step evaluation**, cost & latency per task (DeepEval agent metrics). Use **Langfuse/LangSmith tracing** to _see_ and debug the agent's steps.

### ▶ PROJECT #2 (portfolio) — a multi-step LangGraph agent

Solves a real workflow (e.g., a research assistant that searches → reads → synthesizes → cites), with **tracing + an eval harness measuring task success**.

## Phase 5 — MCP (Model Context Protocol) · Days 20–21 (~7 hrs)

MCP became the industry-standard integration layer in 2025–26 (adopted by Anthropic, OpenAI, Google, Microsoft, AWS; now under the Linux Foundation). Building one is a fresh, high-signal portfolio piece.

- **What/why:** the "USB-C for AI"; it turns the _n×m_ integration problem into _n+m_. Three primitives — **tools, resources, prompts** — over a host/client/server model with stdio or HTTP transports. It's the _integration layer_, not a replacement for good prompting.
- **▶ POC:** build a small **MCP server with FastMCP** exposing a couple of tools/resources (wrap an internal API or a local dataset) and connect it to Claude Desktop / your agent.
- Rewire your Phase-4 agent to reach tools **via MCP** instead of bespoke glue.
- **Security awareness** (interviewers ask): tool poisoning, prompt injection, human-in-the-loop **approval gates**, least privilege.
- **Deliverable:** a working MCP server in your repo + your agent consuming it.

## Phase 6 — Fine-tuning (LoRA / QLoRA) · Days 22–25 (~14 hrs) ★ _POC — kept deliberately scoped_

The point isn't to become a fine-tuning specialist in four days — it's to build **judgment** and one credible artifact.

- **The decision framework first** (most important, and interviewers probe it): fine-tune for **format/style/tone, narrow tasks, latency/cost via smaller models** — _not_ to add knowledge (that's RAG's job). Honestly, well-engineered prompting + RAG wins more often than people expect.
- **Concepts:** full FT vs PEFT; **LoRA** (low-rank adapters) and **QLoRA** (quantized) — how they bring training down to a single GPU; ranks, target modules, key hyperparameters.
- **Data is the hard part:** instruction-dataset curation & formatting, train/val split, quality > quantity.
- **▶ POC (~4 hrs):** LoRA fine-tune a small open model (3–8B) on one narrow task with `transformers + peft + trl`, or **Unsloth** for speed. Use free GPU (Colab/Kaggle) or cheap GPU (Runpod/Modal). _[Axolotl / MLX = alt.]_
- **Evaluate the fine-tune** (this is the lesson): before/after on a held-out set; watch for overfitting/catastrophic forgetting; **compare against a good prompt-only baseline**. If prompting was enough, _say so_ — that's senior thinking.
- _Awareness:_ merging adapters, quantization, vLLM serving.
- **Deliverable:** a fine-tuned adapter + an **eval report comparing it to the prompt baseline**, with your recommendation.

## Phase 7 — Productionization / LLMOps & Guardrails · Days 26–27 (~7 hrs)

Where your existing engineering makes you dangerous. This is what separates "demo builder" from "hire."

- **Serve**: wrap it in FastAPI; async/queue for long agent runs; stream to the client.
- **Deploy**: Docker; a cloud host (Railway/Render/Fly/Modal or AWS); env/secrets management.
- **LLMOps**: prompt versioning, tracing/observability (Langfuse), **cost tracking & budgets**, semantic caching, rate limiting, retries/fallbacks & model routing, latency tuning.
- **Guardrails & safety**: input/output validation (Pydantic), PII handling, **prompt-injection defense**, content moderation, structured-output enforcement.
- **Monitoring in prod**: online evals, drift, feedback loops, **golden-dataset regression gates in CI/CD**.
- **Deliverable:** take one earlier project and make it _production-shaped_ — Dockerized, deployed, traced, cost-tracked, with a CI eval gate.

## Phase 8 — CAPSTONE · Days 28–30+ (~15–20 hrs; may spill a few days) ★ _integrate everything_

One flagship project that pulls the whole month together — the thing you actually talk about in interviews.

- **Should combine:** RAG + a LangGraph agent + tool use **via MCP** + structured outputs + a **full eval harness** (RAGAS + DeepEval + tracing) + deployment (FastAPI/Docker) + guardrails; **optionally** a fine-tuned component.
- **A strong shape:** a domain-specific _agentic assistant_ that retrieves over a real knowledge base, uses tools/MCP to act, is **evaluated end-to-end with metrics in the README**, and is deployed with observability and cost tracking.
- **You mentioned you have a capstone idea — send it to me and I'll turn it into a scoped spec** with milestones, an architecture diagram, and an eval plan sized to your remaining days.
- **Portfolio packaging** (do not skip — this is what gets you interviews): clean READMEs (problem → architecture diagram → **eval metrics** → demo GIF/Loom), a short blog/LinkedIn writeup per project, pin the capstone on GitHub.

---

## Threaded throughout — Interview Prep (you're job-hunting)

Don't leave this to the last day; do a little most days once you're past Phase 2.

- **LLM system design**: design a RAG system, design an agent; talk cost/latency tradeoffs, scaling, failure modes, and — every time — _"how would you evaluate this?"_
- **Concept fluency**: the RAG-vs-fine-tuning-vs-prompting decision framework; causes & mitigations of hallucination; your eval methodology; MCP; agent reliability.
- **DSA**: you're already strong — light maintenance only, for shops that still test it.
- **Talking points**: rehearse each project as a story — the problem, the architecture, and _the eval metric you moved_. The honesty ("we tried fine-tuning; prompting was enough") lands well.

---

## If you fall behind (triage order)

Time is the scarce resource. If a week slips, cut in this order, and protect the starred items:

1. Trim **Phase 6 (fine-tuning)** to the POC + decision framework only.
2. Lighten **Phase 5 (MCP)** to concept + a tiny server.
3. Compress **Phase 7** to "Dockerize + deploy + basic tracing".
4. **Never cut:** Phase 2 (Evaluation), the RAG eval work in Phase 3, the agent eval work in Phase 4, and the Capstone. Those four are your whole differentiation.

## What "done" looks like at the end of the month

- 3 portfolio repos (RAG app, agent, capstone) + an MCP server + a fine-tuning experiment, each with a **metrics-backed README**.
- You can answer "how do you know it's correct, and how do you improve it?" with a real methodology and real numbers.
- A short writeup per project and a clear interview narrative.

_See the companion file `ai_engineer_resources.md` for curated learning resources per phase, and `ai_engineer_roadmap_checklist.pdf` for a printable checkbox version._