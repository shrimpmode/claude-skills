# Docker templates

Substitute `<python>`, `<uv>`, and `<postgres>` with the versions resolved in SKILL.md step 2. Never leave a placeholder unresolved.

## Dockerfile (FastAPI example — Django differs only in the CMDs, noted inline)

Four stages: `base` resolves and installs *all* dependencies including dev tools once
(cacheable, no app code yet); `dev` and `builder` both branch off it with app code added,
diverging only in whether dev tools stay installed; `production` is the minimal runtime image
built from `builder`. `dev` is what `reference/dev-environment.md`'s compose override target
builds — this file doesn't use it directly, but its presence here is why the stages are split
this way instead of the older two-stage builder/final shape.

```dockerfile
FROM python:<python>-slim AS base

COPY --from=ghcr.io/astral-sh/uv:<uv> /uv /uvx /bin/
WORKDIR /app

ENV UV_COMPILE_BYTECODE=1 UV_LINK_MODE=copy

# Dependency layer first (incl. dev tools), so app-code edits don't invalidate it
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-install-project

# --- dev: app code + dev tools, run via the compose override with a bind mount ---
FROM base AS dev
COPY . .
RUN uv sync --frozen
ENV PATH="/app/.venv/bin:$PATH"
EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]
# Django, use instead:
# CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]

# --- builder: app code, dev tools pruned back out (uv sync --no-dev is an exact sync) ---
FROM base AS builder
COPY . .
RUN uv sync --frozen --no-dev

# --- production: minimal runtime, no uv, no dev tools, non-root ---
FROM python:<python>-slim AS production
RUN apt-get update && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*
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
- `base`'s `uv sync --frozen --no-install-project` (no `--no-dev`) installs everything, dev tools included, in one shared, cacheable layer — both `dev` and `builder` start from it.
- `builder`'s `uv sync --frozen --no-dev` after `base` already has dev tools installed still ends up dev-tool-free: `uv sync` is an *exact* sync by default, so `--no-dev` prunes the dev group back out rather than being a no-op. This is what keeps dev tooling out of the `production` image while still sharing `base`'s cache layer with `dev`.
- `dev`'s stage keeps dev tools and uses the framework's own autoreloading dev server (`--reload` for uvicorn, `runserver`'s built-in reloader for Django) instead of gunicorn — gunicorn is a production server and doesn't reload on code changes.
- Only `production` needs the non-root user, `curl`, and the `HEALTHCHECK` — `dev` is a local convenience image, never deployed, so it runs as the default (root) user for simplicity and isn't expected to pass a container security review.
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
    build:
      context: .
      target: production
    env_file: .env
    ports:
      - "8000:8000"
    depends_on:
      db:
        condition: service_healthy

volumes:
  pgdata:
```

`target: production` is required, not decorative, once the Dockerfile has more than one final
stage (`dev`, `production`) — without it, `docker build` builds whichever stage is written last
in the file, silently. This base file is the one used as-is (e.g. in CI, or a real deploy);
`reference/dev-environment.md`'s `docker-compose.override.yml` is what points local `docker
compose up` at the `dev` target instead, and Compose merges the two automatically.

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
