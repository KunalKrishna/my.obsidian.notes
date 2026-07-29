# Applify — OpenClaw Personal Job Application AI Assistant

## Overview

Build **Applify**, a Dockerized, AI-powered job application pipeline that:

1. Ingests job links (career pages, LinkedIn, Indeed, Handshake, etc.)
2. Scrapes job descriptions (with optional credential-based login)
3. Generates tailored resumes + cover letters via OpenAI
4. Tracks every application through a rich dashboard
5. Prevents duplicate applications
6. Lets you retrieve the exact resume used for any interview callback

---

## Security Advisory: Local vs Docker

IMPORTANT

**Recommendation: Docker (isolated bridge network)**

Running inside Docker with a **named volume** (not a bind-mount to your full home folder) gives you:

- No access to your host filesystem beyond the explicitly mounted `~/applify-data` folder
- Process isolation — a rogue dependency can't touch your HDD
- Easy wipe/rebuild (`docker compose down -v`)

**What you lose vs. bare-metal**: nothing meaningful for this use-case. We will mount **only** a single dedicated data directory (`d:\OpenClawJobProject\applify-data`) — the container cannot see anything else on your disk.

---

## Open Questions

IMPORTANT

**Resume Format**: Should the system output resumes as `.docx`, `.pdf`, or both? PDF is better for ATS; `.docx` is easier to manually edit before submitting.

IMPORTANT

**Login Credentials for gated sites (LinkedIn/Handshake)**: Do you want to store credentials inside the app (encrypted in the DB), or provide them once per session via the UI? Session-based is safer.

IMPORTANT

**Batch Review Window**: You mentioned reviewing all applications at 9PM. Should "batch mode" queue jobs silently and block submission until you manually approve, OR auto-submit after your review window even if you're away?

WARNING

**Browser Automation (Playwright)**: Scraping LinkedIn/Handshake requires a real browser session with your cookies. Playwright runs headlessly inside Docker but needs your credentials. Is that acceptable, or do you prefer a manual "paste the job description text" fallback?

NOTE

**OpenAI Model**: Defaulting to `gpt-4o` for resume/cover letter generation. Confirm if you'd prefer `gpt-4o-mini` (cheaper, slightly less quality) or `o3` (more expensive, highest quality).

---

## Tech Stack

|Layer|Technology|Reason|
|---|---|---|
|**Containerization**|Docker + Docker Compose|Security isolation|
|**Backend API**|FastAPI (Python)|Async, fast, OpenAPI docs built-in|
|**AI / LLM**|OpenAI `gpt-4o`|Resume/CL generation, JD parsing|
|**Web Scraping**|Playwright (Python)|Handles JS-heavy pages, login flows|
|**Resume Generation**|`python-docx` + `WeasyPrint`|DOCX editing → PDF export|
|**Database**|PostgreSQL|Robust relational tracking|
|**Frontend**|React + Vite|Rich dashboard UI|
|**Job Queue**|Redis + Celery|Batch processing, async scraping|
|**File Storage**|Named Docker Volume|Isolated from host HDD|
|**Auth (App)**|JWT (local-only, no cloud)|Simple session for the web UI|

---

## Architecture

┌─────────────────────────────────────────────────────────┐

│                    Docker Compose Network                │

│                                                         │

│  ┌──────────┐    ┌──────────────┐    ┌──────────────┐  │

│  │  React   │◄──►│  FastAPI     │◄──►│  PostgreSQL  │  │

│  │  UI      │    │  Backend     │    │  Database    │  │

│  │ :3000    │    │  :8000       │    │  :5432       │  │

│  └──────────┘    └──────┬───────┘    └──────────────┘  │

│                         │                               │

│              ┌──────────┼──────────┐                    │

│              ▼          ▼          ▼                    │

│        ┌─────────┐ ┌────────┐ ┌────────┐               │

│        │Playwright│ │Celery  │ │ Redis  │               │

│        │Scraper  │ │Workers │ │ Queue  │               │

│        └─────────┘ └────────┘ └────────┘               │

│              │                                          │

│              ▼                                          │

│        ┌─────────────────────┐                         │

│        │  OpenAI API (ext.)  │                         │

│        └─────────────────────┘                         │

│                                                         │

│  📁 Named Volume: /data (resumes, master resume)       │

└─────────────────────────────────────────────────────────┘

---

## Proposed Changes (Phase 1 — MVP)

### Phase 1 Scope

1. Job ingestion (URL paste + Playwright scrape with login support)
2. Resume tailoring + Cover Letter via OpenAI
3. Review & approve UI (manual gate before anything is sent)
4. Application tracker dashboard
5. Duplicate detection
6. Resume-to-application linkage (for interview callbacks)
7. Docker Compose setup

---

### Project Structure

d:\OpenClawJobProject\CareerForge\

├── docker-compose.yml

├── .env.example

├── README.md

│

├── backend/                        ← FastAPI service

│   ├── Dockerfile

│   ├── requirements.txt

│   ├── main.py

│   ├── config.py

│   ├── database.py

│   ├── models/

│   │   ├── job.py

│   │   ├── resume.py

│   │   └── application.py

│   ├── routers/

│   │   ├── jobs.py

│   │   ├── resumes.py

│   │   ├── applications.py

│   │   └── auth.py

│   ├── services/

│   │   ├── scraper.py              ← Playwright scraping + login

│   │   ├── ai_engine.py            ← OpenAI resume/CL generation

│   │   ├── resume_builder.py       ← python-docx + PDF export

│   │   ├── duplicate_detector.py   ← Hash/embedding dedup

│   │   └── batch_queue.py          ← Celery task definitions

│   └── playwright_scripts/

│       ├── linkedin_login.py

│       ├── indeed_login.py

│       └── generic_scraper.py

│

├── frontend/                       ← React + Vite dashboard

│   ├── Dockerfile

│   ├── package.json

│   ├── src/

│   │   ├── App.jsx

│   │   ├── pages/

│   │   │   ├── Dashboard.jsx       ← Main tracker table

│   │   │   ├── NewApplication.jsx  ← Paste URL + upload resume

│   │   │   ├── ReviewApprove.jsx   ← Approve generated resume/CL

│   │   │   ├── ResumeVault.jsx     ← All generated resumes

│   │   │   └── Settings.jsx

│   │   └── components/

│   │       ├── JobCard.jsx

│   │       ├── StatusBadge.jsx

│   │       ├── ResumePreview.jsx

│   │       └── StatsBar.jsx

│

├── worker/                         ← Celery worker

│   ├── Dockerfile

│   └── celery_app.py

│

└── postgres/

    └── init.sql                    ← DB schema

---

### Database Schema

sql

-- Jobs table: raw scraped job data

jobs (id, url, company, title, description, source_platform, scraped_at, jd_hash)

-- Resumes table: all resume versions

resumes (id, version_name, file_path, is_master, created_at, tech_stack_tags)

-- Applications table: the core tracker

applications (

  id, job_id, resume_id,

  status,           -- QUEUED | PENDING_REVIEW | APPROVED | APPLIED | REJECTED | INTERVIEW

  applied_at, notes,

  cover_letter_path,

  ai_ats_score      -- estimated ATS match %

)

---

### Key Service Descriptions

#### `scraper.py`

- Detects platform from URL (LinkedIn, Indeed, Handshake, generic)
- Uses Playwright headless browser
- If page is gated: injects stored session cookie OR triggers login flow
- Returns structured `JobDescription` object

#### `ai_engine.py`

- **JD Parser**: Extracts required skills, experience, tech stack from raw text
- **Skill Gap Analyzer**: Compares JD skills vs master resume skills
- **Resume Tailor**: Rewrites/reorganizes bullet points to match JD (does NOT fabricate experience)
- **Cover Letter Generator**: Company-aware, role-specific letter

#### `duplicate_detector.py`

- Primary: URL normalization + exact match
- Secondary: JD embedding similarity (OpenAI embeddings) to catch reposts

#### `resume_builder.py`

- Takes AI-tailored JSON → fills DOCX template → exports PDF
- Stores both versions in named volume `/data/resumes/`

---

## Verification Plan

### Automated

- `docker compose up --build` — all services start cleanly
- FastAPI `/docs` endpoint accessible at `http://localhost:8000/docs`
- Frontend accessible at `http://localhost:3000`

### Manual Verification

- Paste a public job URL → confirm JD is scraped
- Upload master resume → confirm tailored resume is generated
- Review + approve → confirm application record created
- Dashboard shows application with correct resume link
- Re-paste same URL → confirm duplicate warning fires

---

## Phase 2 (Planned — Not Built Now)

|Feature|Tool|
|---|---|
|Job Discovery|SerpAPI Google Jobs + Playwright|
|Discord Bot|`discord.py` — notifications + approval commands|
|Batch Queue UI|Schedule window (e.g., 9PM daily flush)|
|Hosted Deployment|Railway / Render / fly.io|
|Auto-Apply|Playwright form-fill on application pages|

---

## README Contents (to be generated)

- Overall Objective
- Complete Tech Stack table
- Architecture diagram (ASCII + Mermaid)
- OpenClaw component detail
- Features: current + planned roadmap
- Docker setup instructions
- Environment variables reference