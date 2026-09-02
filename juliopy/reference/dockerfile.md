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
RUN groupadd --system app && useradd --system --gid app app
WORKDIR /app
COPY --from=builder --chown=app:app /app /app
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
      - pgdata:/var/lib/postgresql/data
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
