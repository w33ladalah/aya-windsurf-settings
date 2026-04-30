---
trigger: glob
globs: backend/**
---

# Do Not Restart Backend Container

**Never** restart, rebuild, or recreate the backend Docker container when making code changes.

## Why

The backend runs FastAPI with `uvicorn --reload`, which automatically detects file changes and hot-reloads. Restarting the container is unnecessary, slow, and disrupts active WebSocket connections and in-progress AI predictions.

## Forbidden Commands

Do not run any of the following:

- `docker compose restart backend`
- `docker compose up --build backend`
- `docker compose down` / `docker compose up`
- `docker compose stop backend` / `docker compose start backend`
- Any command that causes the backend container to restart

## What To Do Instead

- **Code changes:** Just save the file. Uvicorn will auto-reload within seconds.
- **New pip dependency:** Ask the user to rebuild manually — never do it automatically.
- **Database migrations:** Run `docker compose exec backend alembic upgrade head` (this does not restart the container).
- **Environment variable changes:** These are the only case where a restart is truly needed. Ask the user to restart manually.
