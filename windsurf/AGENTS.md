# AGENTS.md — Aya AI

> Onboarding guide for AI coding agents working on this codebase.

## Project Overview

Aya AI is an AI-powered product image and video generation platform. It is a **stable, production** codebase — prefer minimal, focused changes over broad refactoring.

**Live URLs:**
- Frontend: `https://ayalive.ai`
- Admin: `https://aya-admin.wearepensil.com`
- API: `https://aya-api.wearepensil.com`
- Image proxy: `https://img.ayalive.ai`

## Monorepo Structure

```
aya-ai/
├── backend/        # FastAPI + SQLAlchemy 2.0 (async) + Celery — Python 3.10+
├── frontend/       # Next.js 15 + React 19 + Redux Toolkit + TailwindCSS v4 + MUI 7
├── admin/          # Next.js 16 + React 19 + Redux Toolkit + TailwindCSS v4 + Lucide
├── comfy-toolkit/  # ComfyUI workflow JSONs and helper scripts
├── landing/        # Marketing website (static HTML)
├── docker/         # Container configs for frontend, imgproxy
├── aws/            # Infrastructure configs (CloudFormation)
├── docs/           # Project documentation (see docs/README.md for index)
├── scripts/        # Utility scripts (DB backup, etc.)
├── prompts/        # AI prompt templates
├── .windsurf/      # Windsurf rules, workflows, skills
├── docker-compose.yaml        # Production Docker Compose
└── docker-compose.local.yaml  # Local dev Docker Compose
```

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Backend framework | FastAPI with uvicorn (`--reload` in dev) |
| ORM | SQLAlchemy 2.0 async with asyncpg driver |
| Database | PostgreSQL 15 |
| Migrations | Alembic 1.16+ |
| Task queue | Celery 5.3+ with Redis broker |
| Cache/Broker | Redis 7 |
| Auth | JWT Bearer tokens (python-jose) + bcrypt + Google OAuth |
| File storage | AWS S3 (boto3) |
| Image proxy | Imgproxy (on-the-fly optimization) |
| AI services | Replicate API, ComfyUI |
| Frontend framework | Next.js 15 (App Router) + React 19 |
| Frontend state | Redux Toolkit + redux-persist |
| Frontend HTTP | Axios with interceptors (auto token refresh) |
| Frontend styling | TailwindCSS v4 + MUI 7 |
| Frontend canvas | Konva.js + react-konva |
| Admin framework | Next.js 16 (App Router) + React 19 |
| Admin HTTP | Native `fetch()` with Bearer tokens |
| Admin icons | Lucide React |
| Monitoring | Sentry (backend + frontend) |

## Docker & Environment

All backend services run via Docker Compose. Services: `backend`, `db` (Postgres 15), `redis` (Redis 7), `celery-worker`, `celery-beat`, `adminer`.

```bash
# Local dev (hot-reload volume mounts)
docker compose -f docker-compose.local.yaml up --build

# Production-like
docker compose up --build
```

### Critical Rules

1. **Never restart the backend container** for code changes — uvicorn `--reload` handles it automatically.
2. **Always run backend commands inside Docker:**
   ```bash
   docker compose exec backend <command>
   ```
   Run from project root `/Users/hendrowibowo/Projects/aya-ai`.
3. **Never run `pip install`, `alembic`, or `python` directly on the host.**
4. Frontend and admin run directly on the host: `cd frontend && npm run dev` / `cd admin && npm run dev`.

### Environment Variables

- Root `.env` — shared secrets (AWS, Replicate, Postgres, Redis)
- `backend/.env` — backend-specific config
- `frontend/.env` — `NEXT_PUBLIC_API_BASE_URL`, `NEXT_PUBLIC_GOOGLE_CLIENT_ID`, etc.
- `admin/.env` — `NEXT_PUBLIC_API_URL`, etc.
- Copy `.env.example` → `.env` for new setups. **Never commit `.env` files.**

## Backend Architecture

### Directory Layout

```
backend/
├── main.py                    # FastAPI app, lifespan, CORS, router mounts
├── app/
│   ├── api/__init__.py        # Protected API router aggregator
│   ├── core/
│   │   ├── config.py          # Settings (pydantic-settings, env-driven)
│   │   ├── security.py        # JWT helpers, password hashing, auth dependencies
│   │   ├── celery_app.py      # Celery app + beat schedule
│   │   ├── s3.py              # S3 upload helpers
│   │   └── redis_cache.py     # Redis caching utilities
│   ├── crud/                  # CRUD classes (one per domain entity)
│   ├── db/
│   │   ├── session.py         # AsyncEngine + get_db dependency
│   │   └── redis.py           # RedisManager singleton
│   ├── models/                # SQLAlchemy models (extend BaseModel from base.py)
│   ├── routers/               # FastAPI routers (public + protected)
│   ├── schemas/               # Pydantic v2 request/response schemas
│   ├── tasks/                 # Celery tasks (upload, prompt, photo_model, talking_video, cleanup)
│   └── utils/                 # Shared utilities
├── migrations/                # Alembic migration versions
└── prompts/                   # AI prompt template .txt files
```

### Key Patterns

- **Models:** Extend `BaseModel` from `app/models/base.py` (includes `TimestampMixin`). Use `Mapped[]` + `mapped_column()`. UUIDs as primary keys. Register in `app/models/__init__.py`.
- **CRUD:** One class per entity in `app/crud/`. Constructor takes `db: AsyncSession`. Provide `get_<entity>_crud(db)` dependency at bottom of file.
- **Routers:** Export `router` (protected) and optionally `public_router`. Protected routes register in `app/api/__init__.py`. Public routes mount directly in `main.py`. Admin routes prefix `/api/admin/` with `get_current_active_admin`.
- **Schemas:** Pydantic v2, separate Create/Update/Response models per entity.
- **Auth deps:** `get_current_active_user`, `get_current_active_admin`, `get_current_active_talent`, `get_current_active_superuser`.
- **Settings:** `app/core/config.py` → `Settings` class with DB-backed overrides via `apply_db_overrides()`.
- **Database sessions:** `get_db` yields `AsyncSession` with auto-commit/rollback. Pool: 20 + 10 overflow.

### Database Migrations

```bash
# Generate
docker compose exec backend alembic revision --autogenerate -m "description"

# Apply
docker compose exec backend alembic upgrade head

# Rollback
docker compose exec backend alembic downgrade -1
```

### Image URL Rules

- **Never expose raw S3 URLs** to the frontend. Always use `generate_imgproxy_url()` on the backend.
- **Only S3 URLs** go to imgproxy — never Replicate URLs. If no S3 URL exists, return `None`.
- **Never expose `replicate_url`** in API responses.

### Points System

Dual-currency system: **Green Points** (AI operations) + **Blue Points** (human services).

- Service-based pricing via `ServicePointCost` records.
- Check balance: `check_user_points_for_service()` → HTTP 402 if insufficient.
- Deduct: `deduct_points_for_service()` → creates `PointTransaction` → WebSocket balance update.

### Real-time Communication

- `WebSocketManager` (`app/websocket_manager.py`) handles prediction status, points updates, talking video progress.
- Endpoints: `/api/ws/predictions/{prediction_id}`, `/api/ws/points/{user_id}`.

### Download System

All downloads (images, videos, talking videos) stored in `download_histories` table with `media_type` column:
- `"final"` — FinalImage
- `"photo_model"` — PhotoModelPredictionImage
- `"video"` — VideoGeneration
- `"talking_video"` — TalkingVideo

Flow: ownership check → status check → license resolution → green point deduction → blue point deduction (if talent) → write `DownloadHistory` → mark `is_downloaded=True` → stream bytes.

### Known Bugs / Gotchas

- `photo_model_image_crud.belongs_to_user()` has a UUID vs string comparison bug — always returns `False`. Do a direct inline query instead.
- `Replicate Prediction.wait()` returns `None` and mutates in-place. Never reassign from it.
- `TalkingVideoStatus` enum: `str(video.status)` returns `"TalkingVideoStatus.SUCCEEDED"`, not `"succeeded"`. Compare against enum member directly or use `.value`.

## Frontend Architecture

### Directory Layout

```
frontend/src/
├── app/                 # Next.js App Router pages
│   ├── generate/        # Image generation (Photo Model)
│   ├── edit/            # Image editing (Konva canvas)
│   ├── enhance/         # Image enhancement
│   ├── compose/         # Image composition
│   ├── gallery/         # Public gallery
│   ├── video/           # Video generation
│   ├── talking-video/   # Talking video
│   ├── ai-builder/      # AI Builder
│   ├── quick-tools/     # BG removal, upscale, face swap, cloth try-on, relight
│   ├── user/            # User profile, downloads, favorites
│   └── check-metadata/  # Public metadata checker
├── components/          # Shared React components (flat structure)
├── constants/           # API endpoints (apiEndpoints.ts), static config
├── hooks/               # Custom hooks (useImageProcessing, useImageStatusWebSocket, etc.)
├── providers/           # Context providers (Redux, Error, MUI)
├── services/            # WebSocket service modules
├── store/
│   ├── slices/          # Redux slices (one per domain)
│   ├── thunks/          # Async thunks (one per domain)
│   ├── hooks.ts         # useAppDispatch / useAppSelector
│   └── store.ts         # Store config, RootState, persistor
├── styles/              # Additional CSS
├── types/               # TypeScript types
└── utils/               # Axios instance (api.ts), error handling, image utils
```

### Key Patterns

- **Redux:** One slice + one thunk file per domain. Always use typed hooks (`useAppDispatch`, `useAppSelector`). Persisted slices: `generator`, `auth`, `ui`.
- **API:** Centralized Axios instance in `src/utils/api.ts`. All endpoints in `src/constants/apiEndpoints.ts`. 10-minute timeout for AI operations.
- **Proxy:** `next.config.ts` rewrites `/api/*` → `http://127.0.0.1:8000/api/*` for local dev.
- **Styling:** TailwindCSS v4 primary, MUI for complex components. Font: Nunito.
- **Images:** `unoptimized: true` (imgproxy handles optimization).
- **Auth modal:** Only shown to users with existing `localStorage.refreshToken` (expired sessions). Fresh visitors get no auth interruptions.
- **Header badge:** Green pill on profile avatar when generation succeeds. Non-image generations use `setTalkingVideoSucceeded(1)` dispatch.

### Video Downloads

Use `responseType: "blob"` with axios. Error responses arrive as Blob — must `await data.text()` then `JSON.parse()` to read error detail.

## Admin Architecture

### Directory Layout

```
admin/src/
├── app/                         # Next.js App Router pages (CRUD interfaces)
│   ├── gallery/                 # Gallery management
│   ├── talents/                 # Talent CRUD
│   ├── talent-applications/     # Application review
│   ├── talent-categories/       # Category management
│   ├── talent-featured-images/  # Featured image management
│   ├── pro-refinements/         # Pro refinement management
│   ├── pricing/                 # Point cost management
│   ├── settings/                # App settings
│   ├── static-pages/            # CMS pages (TipTap editor)
│   ├── tips/                    # Tips management
│   ├── users/                   # User management
│   ├── vouchers/                # Voucher management
│   └── clothes-gallery/         # Clothes gallery management
├── components/                  # Shared components (AuthGuard, ReduxProvider)
└── services/                    # API service modules (one per entity)
```

### Key Differences from Frontend

- Next.js 16 (frontend is 15)
- Lucide icons (frontend uses Phosphor + MUI)
- No canvas/Konva editing
- No redux-persist (state resets on reload)
- Native `fetch()` instead of shared Axios instance
- Server-side layout with `Metadata` export
- Font: Geist + Geist Mono

## Testing

- Backend tests in `backend/tests/` (currently limited — `test_face_swap_workflow.py`).
- No frontend/admin test suite currently.
- Interactive API testing via Swagger UI: `http://localhost:8000/docs`.
- **Always ask the user to verify** bug fixes and new features before committing.

## Documentation Requirements

For **new features**, create `docs/<FEATURE_NAME>.md` with: overview, implementation details, files modified, API endpoints, database changes, points cost. Update `docs/README.md`, `docs/API_REFERENCE.md`, and `docs/FEATURES.md`.

For **major bug fixes**, create `docs/<BUG_NAME>_FIX.md` with: problem summary, root cause, solution, files modified. Update `docs/README.md`.

Always check existing docs first — update rather than duplicate.

## Coding Style Rules

1. **Never add or remove comments/documentation** unless explicitly asked.
2. **Maintain existing code style** in every file you touch.
3. **Imports always at the top** — never in the middle of a file.
4. **No emoji in code** unless explicitly requested.
5. **Minimal changes** — this is a stable production codebase. Do not refactor broadly.
6. **Root cause fixes** — prefer upstream fixes over downstream workarounds.

## External Services

| Service | Purpose | Key Config |
|---------|---------|-----------|
| Replicate | AI image generation, upscaling, TTS, video | `REPLICATE_API_TOKEN` |
| ComfyUI | Face swap, BG removal, relighting workflows | `COMFY_URL` |
| AWS S3 | File storage (images, videos, audio) | `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_S3_BUCKET_NAME` |
| MiniMax | Text-to-speech (Speech-02-Turbo) | Via Replicate |
| Google OAuth | Social login | `GOOGLE_CLIENT_ID` |
| Imgproxy | On-the-fly image resizing/optimization | `IMGPROXY_URL`, `IMGPROXY_KEY`, `IMGPROXY_SALT` |
| Sentry | Error tracking | `SENTRY_DSN` |

## Windsurf Configuration

### Rules (`.windsurf/rules/`)

| Rule | Trigger | Purpose |
|------|---------|---------|
| `project-conventions.md` | always_on | Global project rules and structure |
| `backend-conventions.md` | `backend/**` | FastAPI/SQLAlchemy patterns |
| `frontend-conventions.md` | `frontend/**` | Next.js/Redux/MUI patterns |
| `admin-conventions.md` | `admin/**` | Admin panel patterns |
| `do-not-restart-backend.md` | `backend/**` | Never restart backend container |
| `documentation-requirements.md` | always_on | When and how to write docs |
| `verify-before-commit.md` | always_on | Ask user to verify before committing |
| `imgproxy-s3-only.md` | — | Only S3 URLs go to imgproxy |

### Workflows (`.windsurf/workflows/`)

- `/add-new-feature` — Full-stack feature (backend → frontend → admin)
- `/database-migration` — Alembic migration inside Docker
- `/add-api-endpoint` — Schema → CRUD → Router → Register
- `/add-redux-slice` — Redux slice + thunk creation
- `/add-admin-page` — Admin service + page creation
- `/fix-backend-bug` — Backend bug fix workflow
- `/fix-frontend-bug` — Frontend bug fix workflow
- `/fix-admin-bug` — Admin bug fix workflow
- `/review` — Code review workflow
- `/update-docs` — Update docs after feature/fix
- `/save-as-workflow`, `/save-as-skill`, `/save-as-rule`, `/save-as-memory` — Knowledge management
- `/should-save-knowledge` — Decide if a fix is worth saving

### Skills (`.windsurf/skills/`)

- `scaffold-backend-entity` — Generate model, schema, CRUD, router for a new entity
- `scaffold-admin-crud` — Generate admin service + pages for an entity
- `add-celery-task` — Create a new Celery background task
- `add-frontend-page` — Create a frontend page with full Redux wiring
