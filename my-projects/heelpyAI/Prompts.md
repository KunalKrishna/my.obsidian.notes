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

**AI prompt (VS Code):**

> Set up a Python project for a RAG web-scraping pipeline called "heelpy". Create: (1) a docker-compose.yml with a single postgres service using the `pgvector/pgvector:pg16` image, exposing port 5432, with a named volume for persistence; (2) a Python project using uv/venv with a pyproject.toml, and dependencies placeholder for `httpx`, `beautifulsoup4`, `trafilatura`, `psycopg`; (3) an empty sourceURLs.txt file with a comment explaining one URL per line; (4) a .env.example with POSTGRES_USER, POSTGRES_PASSWORD, POSTGRES_DB, DATABASE_URL. Don't write any application logic yet — just the scaffolding.

## Milestone 1 — Bare scraper (print only)

- Script reads `sourceURLs.txt` line by line, fetches each URL (`httpx`/`requests`), extracts clean text (`trafilatura` or `readability-lxml` + BeautifulSoup fallback), prints `url`, `title`, `len(text)`, first 200 chars.
- No crawling, no DB, no recursion — one hop only.
- Handle failures gracefully (timeout, 404, non-HTML content-type) and just log/skip.

**Done when:** you can run it against the seed list and get clean, readable text (not nav/footer boilerplate) for every URL.

**AI prompt (VS Code):**

> Write a Python script scraper/fetch.py that reads URLs from sourceURLs.txt (one per line, skip blank lines and lines starting with #), fetches each URL with httpx (10s timeout, custom User-Agent), and extracts clean article text using trafilatura, falling back to BeautifulSoup if trafilatura returns None. For each URL print: url, title, character count of extracted text, and the first 200 characters. Skip and log (don't crash on) timeouts, non-200 status codes, and non-HTML content types. No database or crawling logic — just single-page fetch and print.

## Milestone 2 — Crawling (still print/JSON only)

- Add link discovery from each fetched page; implement BFS traversal (see note below) with:
    - `max_depth` (start at 3)
    - `url_seen` set on a **normalized** URL (strip fragment, sort query params, lowercase host, strip trailing slash)
    - domain allowlist (only follow links within the same registered domain/subdomain you seeded, or an explicit allowlist of UNC subdomains)
    - a per-domain page budget (e.g. 500) as a safety valve
- Write discovered `(url, depth, parent_url)` to a local JSON/CSV file instead of a DB, so you can eyeball crawl coverage before touching Postgres.
- Respect `robots.txt` disallow rules and add a small delay between requests to the same host.

**Done when:** you can point it at `library.unc.edu` and get a sane, bounded list of real content pages (not thousands of calendar/query-string permutations).

**AI prompt (VS Code):**

> Extend scraper/fetch.py into a crawler (scraper/crawl.py) that starts from the URLs in sourceURLs.txt and does breadth-first traversal up to max_depth=3, following only same-domain (or allowlisted subdomain) links extracted from each page's `<a href>` tags. Normalize URLs before dedup (strip fragment, sort query params, lowercase host, strip trailing slash) and track them in a url_seen set. Respect robots.txt via the urllib.robotparser module, add a 0.5s delay between requests to the same host, and cap each domain at 500 pages. Write results as a JSON list of {url, depth, parent_url, title} to crawl_output.json instead of a database.

## Milestone 3 — Persist to Postgres

- Schema : 
    ```sql
	urls (
		id, url, canonical_url, domain, depth,
		text, text_hash CHAR(32), -- MD5 hex of text  
		http_status, fetched_at, is_active BOOLEAN DEFAULT TRUE
	)
    ```

- Scraper upserts on `canonical_url`; if `text_hash` unchanged from last crawl, skip (no rewrite, no downstream re-embedding trigger later).
- If a previously-seen URL 404s on re-crawl, set `is_active = false` rather than deleting — this is what lets you handle the "event page taken down" case later without breaking source citations for past answers.

**Done when:** re-running the scraper twice back-to-back only updates rows whose content actually changed.

**AI prompt (VS Code):**

> Add PostgreSQL persistence to the crawler. Using psycopg (v3), create a migration/schema.sql that defines a table urls(id serial primary key, url text unique, canonical_url text, domain text, depth int, text text, text_hash char(32), http_status int, fetched_at timestamptz, is_active boolean default true). Modify scraper/crawl.py so that after extracting text for each page it computes an MD5 hash of the text and upserts into the urls table on canonical_url — only overwriting the text/http_status/fetched_at columns if the new hash differs from the stored text_hash. If a previously stored URL now returns a 404, set is_active=false instead of deleting the row. Read DB connection settings from environment variables.

## Milestone 4 — Chunking

- New table `chunks (id, url_id FK, chunk_index, chunk_text, token_count)`.
- Split each `urls.text` into ~300–500 token chunks (recursive character/token splitter with slight overlap, e.g. `langchain-text-splitters` or a hand-rolled version — don't need the rest of LangChain for this).
- Re-chunk only rows whose `text_hash` changed since last run.

**Done when:** chunk counts look reasonable (roughly 1–5 chunks per page depending on length).

**AI prompt (VS Code):**

> Add a chunks table to migration/schema.sql: chunks(id serial primary key, url_id int references urls(id), chunk_index int, chunk_text text, token_count int). Write chunker/chunk.py that selects rows from urls where is_active=true, splits urls.text into overlapping chunks of roughly 300-500 tokens (use tiktoken for token counting, cl100k_base encoding, with ~50 token overlap between chunks), and inserts them into chunks, replacing any existing chunks for a url_id when its text_hash changed since the last chunking run (track a text_hash column on chunks or compare against urls.text_hash via join).

## Milestone 5 — Embeddings + pgvector

- Enable `CREATE EXTENSION vector;` on the Postgres DB.
- `chunk_embeddings (chunk_id FK, embedding VECTOR(N))`.
- Start with a **free, local** embedding model via Ollama in Docker — `nomic-embed-text` (768-dim, Apache-2.0, strong on short/long text, ~274MB) is a solid default.
- Batch-embed all chunks, add an HNSW index (`USING hnsw (embedding vector_cosine_ops)`) — HNSW is the right default at this scale, no need for IVFFlat.

**Done when:** `SELECT` with `<=>` cosine-distance ordering returns something for a hand-typed query vector.

**AI prompt (VS Code):**

> Set up local embeddings. Add an Ollama service to docker-compose.yml (image ollama/ollama, port 11434, volume for model cache) and a one-time setup command to pull the nomic-embed-text model. Add a chunk_embeddings table to schema.sql: chunk_embeddings(chunk_id int primary key references chunks(id), embedding vector(768)). Enable the pgvector extension in the migration. Write embed/embed.py that batches unembedded chunks, calls the Ollama /api/embeddings endpoint with model=nomic-embed-text for each chunk_text, and inserts the resulting vector into chunk_embeddings. After the initial backfill, create an HNSW index: CREATE INDEX ON chunk_embeddings USING hnsw (embedding vector_cosine_ops).

## Milestone 6 — Retrieval-only test (no LLM yet)

- Standalone script: hardcode a query like _"free groceries on campus"_, embed it with the same model used for indexing, run top-k similarity search, print the retrieved chunk text + source URL.
- Manually eyeball 10–15 test queries against retrieval quality before adding any generation step. This isolates retrieval bugs from LLM bugs.

**Done when:** for a handful of known questions, the right page shows up in the top 3 results.

**AI prompt (VS Code):**

> Write retrieval/search.py with a function search(query: str, top_k: int = 5) that embeds the query using the same Ollama nomic-embed-text model, then runs a SQL query against chunk_embeddings joined to chunks and urls, ordering by embedding <=> query_embedding (cosine distance) and returning the top_k rows with chunk_text, url, and title. Add a **main** block that takes a hardcoded list of 10 test questions (e.g. "free groceries on campus", "quantum computing course", "fee payment deadline"), runs search on each, and pretty-prints the top 3 results with their source URLs for manual review.

## Milestone 7 — LLM integration (CLI)

- Wire retrieved chunks + user query into a prompt template that instructs the model to answer only from context and cite source URLs.
- Start with a local model via Ollama (e.g. `llama3.1:8b` or `qwen2.5:7b`) to keep the whole stack free end-to-end; swap in Claude/OpenAI later via an env-var-driven adapter so the switch is a config change, not a rewrite.
- Output: answer text + list of source URLs actually used.

**Done when:** a CLI script takes a question and returns an answer with correct citations for your test set.

**AI prompt (VS Code):**

> Write rag/answer.py that takes a user question, calls retrieval/search.search() to get top_k chunks, builds a prompt instructing the model to answer only using the provided context and to list which source URLs it used (or say it doesn't know if the context is insufficient), and sends it to a local Ollama chat model (llama3.1:8b) via the /api/chat endpoint. Structure the model call behind a small LLMClient interface with an OllamaClient implementation now and a stubbed AnthropicClient/OpenAIClient for later, selected via an LLM_PROVIDER environment variable. Add a **main** CLI that takes a question as a command-line arg and prints the answer plus a "Sources:" list of URLs.

## Milestone 8 — REST API

- FastAPI app: `POST /ask`, `GET /health`, `GET /sources/{url_id}`.
- Keep the retrieval + generation logic as a plain importable module so the API is a thin wrapper — you'll want to reuse it in Milestone 10's scheduled job too.

**AI prompt (VS Code):**

> Wrap rag/answer.py and retrieval/search.py in a FastAPI app (api/main.py) with endpoints: POST /ask accepting {question: str} and returning {answer: str, sources: [{url: str, title: str}]}; GET /health returning {status: "ok"}; GET /sources/{url_id} returning the stored urls row. Add CORS middleware allowing the React dev server origin. Add a Dockerfile for the API service and add it to docker-compose.yml.

## Milestone 9 — Frontend

- Minimal React single-page chat UI: input box, answer display, clickable source links below the answer.
- No auth/accounts needed yet — this is a public Q&A tool.

**AI prompt (VS Code):**

> Create a minimal React app (Vite + plain CSS, no UI framework) with a single chat page: a text input and submit button, a scrollable list of past Q&A turns, and for each answer a "Sources" section listing clickable links. On submit, POST to the FastAPI /ask endpoint (base URL from an env var) and render the returned answer and sources. Keep it to a handful of files — App.jsx, a ChatMessage component, and a simple api.js client.

## Milestone 10 — Freshness & scheduling

- Cron/Docker scheduled job re-runs the crawler on a schedule (e.g. nightly for high-churn subdomains like events/news, weekly for static ones like registrar policy pages).
- Hash-diffing (already built in M3/M4) means only changed pages get re-chunked/re-embedded — keeps this cheap.
- Surface `is_active = false` sources distinctly in answers (e.g. "this page may no longer be available") instead of silently dropping them.

**AI prompt (VS Code):**

> Add a scheduled job scraper/rerun.py that re-runs crawl.py, chunk.py, and embed.py in sequence for a given list of domains, intended to be invoked by cron or a Docker Compose "scheduler" service. Add two cron schedules in a crontab file: nightly for high-churn domains (news/events subdomains) and weekly for the rest, read from a config file mapping domain -> schedule tier. Update api/main.py's /ask response so that if a returned source has is_active=false, the source object includes a stale: true flag, and update the React ChatMessage component to render stale sources with a "may no longer be available" label.

## Milestone 11 — Production hardening

- Swap local models for paid APIs: OpenAI/Voyage/Claude embeddings + Claude/GPT for generation (config-driven, per M7).
- Build a small eval set (30–50 question → expected-source pairs) and re-run it after every model/pipeline change to catch regressions.
- Add rate limiting, basic auth if needed, structured logging, and monitoring on crawl failures.

**AI prompt (VS Code):**

> Add AnthropicClient and OpenAIClient implementations of the LLMClient interface from rag/answer.py, and an equivalent alternate embedding client for Voyage AI/OpenAI embeddings, both selected via environment variables (LLM_PROVIDER, EMBEDDING_PROVIDER) so switching providers requires no code changes. Write eval/eval.py that loads a JSON file of {question, expected_url_substrings} test cases, runs each through the full RAG pipeline, and reports for each case whether any returned source URL matches an expected substring, plus an overall pass rate — intended to be re-run after any pipeline or model change to catch regressions.

### Parking lot (not blocking, revisit later)

Hybrid search (BM25 + vector), a reranker stage, query rewriting/expansion, an admin UI for managing `sourceURLs.txt`, PDF ingestion (course catalogs, forms), and multi-turn conversation memory.