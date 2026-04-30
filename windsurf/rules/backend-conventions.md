---
trigger: glob
globs: backend/**
---

# Backend Conventions (FastAPI + Python)

## Tech Stack
- **Framework:** FastAPI 0.100+ with uvicorn
- **ORM:** SQLAlchemy 2.0 (async) with asyncpg
- **Migrations:** Alembic 1.16+
- **Task queue:** Celery 5.3+ with Redis broker
- **Auth:** JWT (python-jose) + bcrypt + Google OAuth
- **Storage:** AWS S3 via boto3
- **AI services:** Replicate API, ComfyUI
- **Monitoring:** Sentry SDK
- **Config:** pydantic-settings (`app/core/config.py` → `Settings` class)

## Directory Layout
```
backend/
├── main.py                  → FastAPI app, lifespan, CORS, router mounts
├── app/
│   ├── api/__init__.py      → Protected API router aggregator
│   ├── core/
│   │   ├── config.py        → Settings (pydantic-settings, env-driven)
│   │   ├── security.py      → JWT helpers, password hashing, auth dependencies
│   │   ├── celery_app.py    → Celery app + beat schedule
│   │   ├── s3.py            → S3 upload helpers
│   │   └── redis_cache.py   → Redis caching utilities
│   ├── crud/                → CRUD classes (one per domain entity)
│   ├── db/
│   │   ├── session.py       → AsyncEngine + get_db dependency
│   │   └── redis.py         → RedisManager singleton
│   ├── models/              → SQLAlchemy models inheriting BaseModel
│   ├── routers/             → FastAPI routers (public + protected)
│   ├── schemas/             → Pydantic v2 request/response schemas
│   ├── tasks/               → Celery tasks
│   └── utils/               → Shared utilities
├── migrations/              → Alembic migration versions
├── prompts/                 → AI prompt template files
└── workflows/               → ComfyUI workflow JSONs
```

## Patterns & Conventions

### Models
- All models extend `BaseModel` from `app/models/base.py` (includes `TimestampMixin` with `created_at`/`updated_at`).
- Use `Mapped[]` + `mapped_column()` (SQLAlchemy 2.0 style).
- UUIDs as primary keys: `UUID(as_uuid=True), default=uuid.uuid4`.
- Relationships use `lazy="raise"` by default (explicit loading required).
- Register all models in `app/models/__init__.py`.

### CRUD
- CRUD classes take `db: AsyncSession` in `__init__`.
- Provide a `get_<entity>_crud(db)` dependency function at the bottom of each file.
- Use `select()` from `sqlalchemy.future` for queries.

### Routers
- Public routes: mount directly on `app` in `main.py` with explicit prefix.
- Protected routes: register in `app/api/__init__.py` under `api_router` (auto-protected by `get_current_active_user` dependency).
- Admin routes: prefix with `/api/admin/`, use `get_current_active_admin` dependency.
- Router files export `router` (protected) and optionally `public_router`.

### Schemas
- Use Pydantic v2 (`BaseModel` from pydantic).
- Separate Create/Update/Response schemas per entity.

### Auth
- JWT Bearer tokens stored client-side (`localStorage`).
- Roles: `get_current_active_user`, `get_current_active_admin`, `get_current_active_talent`, `get_current_active_superuser`.
- Google OAuth via `google-auth` library.

### Celery Tasks
- Registered in `app/core/celery_app.py` `include` list.
- Task files in `app/tasks/`.
- Beat schedule configured in `celery_app.py`.

### Database Sessions
- Use `get_db` dependency → yields `AsyncSession` with auto-commit on success, rollback on error.
- Pool: 20 connections + 10 overflow.

### Running Commands
- **Always inside Docker:** `docker compose exec backend <command>`
- Alembic: `docker compose exec backend alembic revision --autogenerate -m "description"` then `docker compose exec backend alembic upgrade head`
- Never run `pip install`, `alembic`, or `python` scripts directly on the host.

### Settings
- All config via `app/core/config.py` `Settings` class.
- DB-backed overrides via `apply_db_overrides()` — sensitive keys in `DB_OVERRIDE_DENYLIST` are never overridden from DB.
- Access via the singleton: `from app.core.config import settings`.
