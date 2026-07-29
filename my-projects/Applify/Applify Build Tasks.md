# Applify Build Tasks

## Infrastructure

- `[x]` `docker-compose.yml`
- `[x]` `.env.example`
- `[x]` `postgres/init.sql`
- `[x]` `README.md`

## Backend

- `[x]` `backend/Dockerfile`
- `[x]` `backend/requirements.txt`
- `[x]` `backend/config.py`
- `[x]` `backend/database.py`
- `[x]` `backend/main.py`
- `[x]` `backend/models/` (job, resume, application, profile)
- `[x]` `backend/services/eligibility_checker.py`
- `[x]` `backend/services/scraper.py`
- `[x]` `backend/services/ai_engine.py`
- `[x]` `backend/services/resume_builder.py`
- `[x]` `backend/services/auto_apply.py`
- `[x]` `backend/services/duplicate_detector.py`
- `[x]` `backend/routers/` (jobs, resumes, applications, profile)
- `[x]` `backend/data/user_profile.template.json`

## Worker

- `[x]` `worker/Dockerfile`
- `[x]` `worker/celery_app.py`

## Frontend

- `[x]` `frontend/Dockerfile`
- `[x]` `frontend/package.json` + `vite.config.js` + `index.html`
- `[x]` `frontend/src/main.jsx` + `App.jsx` + `index.css`
- `[x]` `frontend/src/pages/Dashboard.jsx`
- `[x]` `frontend/src/pages/NewApplication.jsx`
- `[x]` `frontend/src/pages/ReviewApprove.jsx`
- `[x]` `frontend/src/pages/ResumeVault.jsx`
- `[x]` `frontend/src/pages/Settings.jsx`
- `[x]` `frontend/src/components/` (JobCard, StatusBadge, StatsBar, Sidebar)
- `[x]` `frontend/src/api/client.js`