---
trigger: always_on
---

# Aya AI — Project Conventions

## Project Overview
Aya AI is an AI-powered product image generation platform. It is a **stable, production** codebase.

## Monorepo Structure
```
aya-ai/
├── backend/       → FastAPI + SQLAlchemy + Celery (Python 3.10+)
├── frontend/      → Next.js 15 + React 19 + Redux Toolkit + TailwindCSS v4 + MUI 7
├── admin/         → Next.js 16 + React 19 + Redux Toolkit + TailwindCSS v4 + Lucide icons
├── comfy-toolkit/ → ComfyUI workflow JSONs
├── landing/       → Marketing website
├── docker/        → Container configs
├── aws/           → Infrastructure configs
└── scripts/       → Utility scripts
```

## Docker & Environment
- All services run via `docker-compose.yaml` (prod) or `docker-compose.local.yaml` (dev).
- Services: `backend`, `db` (Postgres 15), `redis` (Redis 7), `celery-worker`, `celery-beat`, `adminer`.
- **Always run backend commands (alembic, python scripts) inside the Docker container:**
  ```bash
  docker compose exec backend <command>
  ```
  Run from project root `/Users/hendrowibowo/Projects/aya-ai`.
- **Never restart the backend container** when making code changes — it uses `--reload` via uvicorn.

## Environment Variables
- Root `.env` holds shared secrets (AWS, Replicate, Postgres, Redis).
- Each sub-project (`backend/.env`, `frontend/.env`, `admin/.env`) has its own env file.
- Copy `.env.example` → `.env` for new setups. Never commit `.env` files.

## Key Domains / URLs
- **Frontend:** `https://ayalive.ai` / `https://aya.wearepensil.com`
- **Admin:** `https://aya-admin.wearepensil.com`
- **API:** `https://aya-api.wearepensil.com`
- **Image proxy:** `https://img.ayalive.ai`

## Git & Stability
- Project status is **stable**. Prefer minimal, focused changes.
- Do not refactor broadly unless explicitly asked.
- Prefer bug fixes at the root cause, not downstream workarounds.

## General Coding Rules
- Do not add or remove comments/documentation unless explicitly asked.
- Maintain existing code style in every file you touch.
- Imports always at the top of the file — never in the middle.
- No emoji in code unless the user explicitly requests it.
