# Docker templates

Substitute `<python>`, `<uv>`, and `<postgres>` with the versions resolved in SKILL.md step 2. Never leave a placeholder unresolved.

## Dockerfile (FastAPI example — Django differs only in the CMD)

```dockerfile
FROM python:<python>-slim AS builder

COPY --from=ghcr.io/astral-sh/uv:<uv> /uv /uvx /bin/
WORKDIR /app

ENV UV_COMPILE_BYTECODE=1 UV_LINK_MODE=copy

# Dependency layer first, so app-code edits don't invalidate it
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-install-project --no-dev

COPY . .
RUN uv sync --frozen --no-dev

FROM python:<python>-slim
RUN groupadd --system app && useradd --system --gid app --home /app app
WORKDIR /app
COPY --from=builder --chown=app:app /app /app
RUN chown app:app /app
ENV PATH="/app/.venv/bin:$PATH"
USER app

EXPOSE 8000
HEALTHCHECK --interval=10s --timeout=3s --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1

# FastAPI:
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
# Django, use instead:
# CMD ["gunicorn", "config.wsgi:application", "--bind", "0.0.0.0:8000"]
```

Notes:
- `curl` must be installed in the final stage (`apt-get install -y --no-install-recommends curl` before dropping to non-root) for the `HEALTHCHECK` to work — or check via a Python one-liner instead if you'd rather not add curl.
- `--no-install-project` on the first `uv sync` installs only dependencies, keeping that layer cacheable independent of app-code changes.
- `useradd --home /app app` gives the non-root user a writable `$HOME`. Without it, servers that write runtime state there (gunicorn's control socket, various caches) fail with `Permission denied` errors in the logs even though the container looks "up" and the healthcheck can still pass — don't mistake a clean `docker compose up` for a clean log.
- `COPY --chown=app:app` only chowns the files it copies in; the `/app` directory entry itself was created by `WORKDIR` while still root, and stays root-owned (`drwxr-xr-x root root`) unless you chown it explicitly. The `RUN chown app:app /app` line above is required for that reason, not redundant with `--chown`.

## docker-compose.yml

```yaml
services:
  db:
    image: postgres:<postgres>
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - pgdata:/var/lib/postgresql  # postgres 18+; use /var/lib/postgresql/data instead on postgres <18
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 5s
      timeout: 3s
      retries: 5

  app:
    build: .
    env_file: .env
    ports:
      - "8000:8000"
    depends_on:
      db:
        condition: service_healthy

volumes:
  pgdata:
```

Postgres 18 changed the image's internal layout to be `pg_ctlcluster`-compatible and expects a single mount at `/var/lib/postgresql` (it then places versioned data in a subdirectory); mounting the old `/var/lib/postgresql/data` path on a 18+ image fails fast on startup with a fatal "these Docker images are configured to store database data in a format which is compatible with pg_ctlcluster" error. Match the mount to the major version resolved in SKILL.md step 2 — don't default to the old path from habit.

## .dockerignore

```
.venv
__pycache__
*.pyc
.git
.env
.pytest_cache
.mypy_cache
.ruff_cache
```
