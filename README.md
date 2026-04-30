# Aya AI — Windsurf Configuration

This repository contains Windsurf AI coding assistant configuration for the Aya AI project.

## About Aya AI

Aya AI is an AI-powered product image generation platform. It is a stable, production codebase built as a monorepo with the following structure:

```text
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

## Repository Structure

This Windsurf configuration contains:

- **`rules/`** — Project conventions and coding standards
- **`skills/`** — Reusable coding patterns and scaffolding templates
- **`workflows/`** — Step-by-step guides for common development tasks

## Rules

The rules define how the AI should interact with the Aya AI codebase:

- `project-conventions.md` — Overall project structure, Docker usage, environment variables
- `backend-conventions.md` — Backend-specific coding standards
- `frontend-conventions.md` — Frontend-specific coding standards
- `admin-conventions.md` — Admin panel coding standards
- `documentation-requirements.md` — Documentation guidelines
- `do-not-restart-backend.md` — Important note about backend hot-reloading

## Skills

Skills are reusable patterns for generating code:

- `scaffold-backend-entity` — Generate complete backend entity (model, schema, CRUD, router)
- `scaffold-admin-crud` — Generate admin CRUD pages
- `add-frontend-page` — Generate frontend pages with Redux integration
- `add-celery-task` — Generate Celery background tasks

## Workflows

Workflows provide step-by-step guidance for common tasks:

- `add-new-feature.md` — End-to-end full-stack feature development
- `add-api-endpoint.md` — Adding new API endpoints
- `add-admin-page.md` — Creating admin panel pages
- `add-redux-slice.md` — Adding Redux state management
- `database-migration.md` — Running Alembic migrations
- `fix-backend-bug.md` — Backend debugging workflow
- `fix-frontend-bug.md` — Frontend debugging workflow
- `fix-admin-bug.md` — Admin panel debugging workflow

## Key Development Guidelines

### Docker & Environment

- All services run via `docker-compose.yaml` (prod) or `docker-compose.local.yaml` (dev)
- Services: `backend`, `db` (Postgres 15), `redis` (Redis 7), `celery-worker`, `celery-beat`, `adminer`
- Always run backend commands inside the Docker container: `docker compose exec backend <command>`
- Never restart the backend container when making code changes — it uses `--reload` via uvicorn

### Environment Variables

- Root `.env` holds shared secrets (AWS, Replicate, Postgres, Redis)
- Each sub-project has its own env file (`backend/.env`, `frontend/.env`, `admin/.env`)
- Copy `.env.example` → `.env` for new setups. Never commit `.env` files

### Git & Stability

- Project status is **stable**. Prefer minimal, focused changes
- Do not refactor broadly unless explicitly asked
- Prefer bug fixes at the root cause, not downstream workarounds

### General Coding Rules

- Do not add or remove comments/documentation unless explicitly asked
- Maintain existing code style in every file you touch
- Imports always at the top of the file — never in the middle
- No emoji in code unless the user explicitly requests it

## Key Domains / URLs

- **Frontend:** `https://ayalive.ai` / `https://aya.wearepensil.com`
- **Admin:** `https://aya-admin.wearepensil.com`
- **API:** `https://aya-api.wearepensil.com`
- **Image proxy:** `https://img.ayalive.ai`

## Usage

This configuration is automatically loaded by Windsurf when working on the Aya AI project. The rules, skills, and workflows guide the AI assistant in:

1. Following project conventions
2. Generating code that matches the existing patterns
3. Executing common development workflows correctly
4. Understanding the monorepo structure and dependencies

## Note

The actual Aya AI codebase is located at `/Users/hendrowibowo/Projects/aya-ai`. This repository only contains the Windsurf configuration.
