# heelpy — UNC-Chapel Hill RAG Assistant: Build Roadmap

## Architecture recap

```
sourceURLs.txt → Scraper (crawl + extract) → Postgres (raw text + hash)
                                                    │
                                                    ▼
                                          Chunker → Embedder → pgvector
                                                    │
                                    Query → embed → similarity search → chunks
                                                    │
                                          LLM (context + query) → answer + source URLs
                                                    │
                                          FastAPI ← → React frontend
```

Both Postgres and pgvector live in the same Docker Postgres instance (pgvector is a Postgres extension, not a separate service). The RDBMS `text` column is the source of truth; the vector table is a derived, fully rebuildable cache.

---

## Milestone 0 — Environment scaffolding

- `docker-compose.yml` with a single `postgres` service (image `pgvector/pgvector:pg16` or similar) — no app code yet.
- Python project skeleton (`uv`/`venv`), `sourceURLs.txt` with 5–10 seed URLs across a few subdomains (library, registrar, dining).
- Check `robots.txt` and `sitemap.xml` for each seed domain; note which domains publish a sitemap (use it later — it's a shortcut around crawling).

**Done when:** `docker compose up` gives you a running empty Postgres you can `psql` into.

## Milestone 1 — Bare scraper (print only)

- Script reads `sourceURLs.txt` line by line, fetches each URL (`httpx`/`requests`), extracts clean text (`trafilatura` or `readability-lxml` + BeautifulSoup fallback), prints `url`, `title`, `len(text)`, first 200 chars.
- No crawling, no DB, no recursion — one hop only.
- Handle failures gracefully (timeout, 404, non-HTML content-type) and just log/skip.

**Done when:** you can run it against the seed list and get clean, readable text (not nav/footer boilerplate) for every URL.

## Milestone 2 — Crawling (still print/JSON only)

- Add link discovery from each fetched page; implement BFS traversal (see note below) with:
    - `max_depth` (start at 3)
    - `url_seen` set on a **normalized** URL (strip fragment, sort query params, lowercase host, strip trailing slash)
    - domain allowlist (only follow links within the same registered domain/subdomain you seeded, or an explicit allowlist of UNC subdomains)
    - a per-domain page budget (e.g. 500) as a safety valve
- Write discovered `(url, depth, parent_url)` to a local JSON/CSV file instead of a DB, so you can eyeball crawl coverage before touching Postgres.
- Respect `robots.txt` disallow rules and add a small delay between requests to the same host.

**Done when:** you can point it at `library.unc.edu` and get a sane, bounded list of real content pages (not thousands of calendar/query-string permutations).

## Milestone 3 — Persist to Postgres

- Schema:
    
    ```sql
    urls (  id, url, canonical_url, domain, depth,  text, text_hash CHAR(32),      -- MD5 hex of text  http_status, fetched_at, is_active BOOLEAN DEFAULT TRUE)
    ```
    
- Scraper upserts on `canonical_url`; if `text_hash` unchanged from last crawl, skip (no rewrite, no downstream re-embedding trigger later).
- If a previously-seen URL 404s on re-crawl, set `is_active = false` rather than deleting — this is what lets you handle the "event page taken down" case later without breaking source citations for past answers.

**Done when:** re-running the scraper twice back-to-back only updates rows whose content actually changed.

## Milestone 4 — Chunking

- New table `chunks (id, url_id FK, chunk_index, chunk_text, token_count)`.
- Split each `urls.text` into ~300–500 token chunks (recursive character/token splitter with slight overlap, e.g. `langchain-text-splitters` or a hand-rolled version — don't need the rest of LangChain for this).
- Re-chunk only rows whose `text_hash` changed since last run.

**Done when:** chunk counts look reasonable (roughly 1–5 chunks per page depending on length).

## Milestone 5 — Embeddings + pgvector

- Enable `CREATE EXTENSION vector;` on the Postgres DB.
- `chunk_embeddings (chunk_id FK, embedding VECTOR(N))`.
- Start with a **free, local** embedding model via Ollama in Docker — `nomic-embed-text` (768-dim, Apache-2.0, strong on short/long text, ~274MB) is a solid default.
- Batch-embed all chunks, add an HNSW index (`USING hnsw (embedding vector_cosine_ops)`) — HNSW is the right default at this scale, no need for IVFFlat.

**Done when:** `SELECT` with `<=>` cosine-distance ordering returns something for a hand-typed query vector.

## Milestone 6 — Retrieval-only test (no LLM yet)

- Standalone script: hardcode a query like _"free groceries on campus"_, embed it with the same model used for indexing, run top-k similarity search, print the retrieved chunk text + source URL.
- Manually eyeball 10–15 test queries against retrieval quality before adding any generation step. This isolates retrieval bugs from LLM bugs.

**Done when:** for a handful of known questions, the right page shows up in the top 3 results.

## Milestone 7 — LLM integration (CLI)

- Wire retrieved chunks + user query into a prompt template that instructs the model to answer only from context and cite source URLs.
- Start with a local model via Ollama (e.g. `llama3.1:8b` or `qwen2.5:7b`) to keep the whole stack free end-to-end; swap in Claude/OpenAI later via an env-var-driven adapter so the switch is a config change, not a rewrite.
- Output: answer text + list of source URLs actually used.

**Done when:** a CLI script takes a question and returns an answer with correct citations for your test set.

## Milestone 8 — REST API

- FastAPI app: `POST /ask`, `GET /health`, `GET /sources/{url_id}`.
- Keep the retrieval + generation logic as a plain importable module so the API is a thin wrapper — you'll want to reuse it in Milestone 10's scheduled job too.

## Milestone 9 — Frontend

- Minimal React single-page chat UI: input box, answer display, clickable source links below the answer.
- No auth/accounts needed yet — this is a public Q&A tool.

## Milestone 10 — Freshness & scheduling

- Cron/Docker scheduled job re-runs the crawler on a schedule (e.g. nightly for high-churn subdomains like events/news, weekly for static ones like registrar policy pages).
- Hash-diffing (already built in M3/M4) means only changed pages get re-chunked/re-embedded — keeps this cheap.
- Surface `is_active = false` sources distinctly in answers (e.g. "this page may no longer be available") instead of silently dropping them.

## Milestone 11 — Production hardening

- Swap local models for paid APIs: OpenAI/Voyage/Claude embeddings + Claude/GPT for generation (config-driven, per M7).
- Build a small eval set (30–50 question → expected-source pairs) and re-run it after every model/pipeline change to catch regressions.
- Add rate limiting, basic auth if needed, structured logging, and monitoring on crawl failures.

### Parking lot (not blocking, revisit later)

Hybrid search (BM25 + vector), a reranker stage, query rewriting/expansion, an admin UI for managing `sourceURLs.txt`, PDF ingestion (course catalogs, forms), and multi-turn conversation memory.