---
name: juliopy
description: Scaffold a production-grade Python backend service — Docker, Postgres, Django or FastAPI, uv, ruff, mypy, pytest, CI, pre-commit, typed config, health checks — always on current latest-stable versions. Use when starting a new Python backend/API project, containerizing a Python service, or setting up a professional Django/FastAPI + Postgres stack.
---

# juliopy

Production-grade stack for Python backends: **Python + uv + Docker + Postgres**, framework is **Django or FastAPI**, quality gates are **ruff + mypy + pytest + pre-commit + CI**, versions are always **latest stable** — never a version remembered from training data, since that goes stale. Resolve real current versions at run time, every time.

Detailed templates live in `reference/` and are loaded per step below — read each file only when you reach its step, not up front.

## Steps

### 1. Pick the framework

Ask only if genuinely ambiguous; otherwise decide from the shape of the ask and say which you picked and why:

- **Django** — needs an admin panel, built-in auth/ORM/migrations, a full-featured monolith, or the user says "batteries included."
- **FastAPI** — a lean API/microservice, async-heavy, OpenAPI/schema-first, or the user says "just an API."

### 2. Resolve latest stable versions — do not guess

Look each of these up before writing any file; do not reuse a version number from memory. Use the PyPI JSON API (`https://pypi.org/pypi/<pkg>/json`, field `info.version`) for Python packages, and Docker Hub tags for images:

- **Python** (python.org/downloads or `python` Docker Hub tags)
- **uv**
- **Framework**: Django or FastAPI, plus its DB driver (`psycopg[binary]` for Django/sync, `asyncpg` for FastAPI/async) and ASGI/WSGI server (`gunicorn`, `uvicorn`). Django also needs `dj-database-url` (parses `DATABASE_URL` into `DATABASES`) and `django-stubs` (mypy plugin, see step 5).
- **Postgres** (Docker Hub `postgres` tags — pin an explicit version, never `latest`; note the major version, step 4's volume mount depends on it)
- **ruff, mypy, pytest** (+ `pytest-asyncio` for FastAPI, `pytest-django` for Django — pytest can't run Django tests/fixtures without it), **pydantic-settings**
- **pre-commit**, and the exact tags of the `astral-sh/ruff-pre-commit` and `pre-commit/mirrors-mypy` repos

### 3. Initialize with uv

Use `uv init --app --no-package .` (not bare `uv init`) — the default creates a `src/`-layout installable package (`[build-system]`, `[project.scripts]`), which doesn't fit a Django/FastAPI service. `--app --no-package` gives a flat layout with no build backend.

Then `uv add` the framework/driver/server packages (+ `dj-database-url` for Django) and `uv add --dev ruff mypy pytest pre-commit` (+ `pytest-asyncio` for FastAPI, or `django-stubs` + `pytest-django` for Django), all pinned to the versions from step 2. This produces `pyproject.toml` and `uv.lock`.

### 4. Scaffold Docker

Read `reference/dockerfile.md` and apply its multi-stage, non-root, uv-based `Dockerfile`, `docker-compose.yml` (app + Postgres, healthchecks, named volume), and `.dockerignore`, substituting the versions from step 2. Also add `.env` to `.gitignore` (create the file if `uv init` didn't) — the app skeleton in step 7 writes real secrets into `.env`, and it must never be committed.

Postgres 18+ images changed their data-directory convention (volume mounts at `/var/lib/postgresql`, not `.../data`) and the non-root Docker user needs an explicit writable `$HOME` — both are easy to get wrong silently; `reference/dockerfile.md` has the corrected template.

### 5. Configure tooling

Read `reference/tooling.md` and add its `[tool.ruff]`, `[tool.mypy]`, and `[tool.pytest.ini_options]` blocks to `pyproject.toml`.

### 6. Wire CI and pre-commit

Read `reference/ci-and-hooks.md` and write `.github/workflows/ci.yml` and `.pre-commit-config.yaml`, including the `gitleaks` secret-scanning hook. Then run `uv run pre-commit install`.

The mypy pre-commit hook runs in its own isolated virtualenv, separate from `uv.lock` — `additional_dependencies` is the *only* thing installed into it. If your settings module imports the framework, DB driver, or settings library (it does), those must be listed there too, pinned to the same versions, not just the `*-stubs` package — otherwise mypy fails with import errors that look unrelated to your code (e.g. `ImproperlyConfigured: Error loading psycopg2 or psycopg module`). `reference/ci-and-hooks.md` has the full list for Django.

### 7. Build the app skeleton

Read `reference/app-skeleton.md` for the chosen framework: a `pydantic-settings` `Settings` class reading env/`.env`, DB session/connection wiring, and `GET /health` (liveness, no I/O) + `GET /ready` (checks Postgres). Point the Docker healthcheck at `/health`.

For Django, run `uv run ruff check --fix . && uv run ruff format .` right after `django-admin startproject`/`startapp` — the generated boilerplate itself (single-quoted strings, unused imports in the stub `admin.py`/`models.py`/`tests.py`) fails `ruff check`/`ruff format` as shipped. Do this before layering in your own code, so step 8 isn't surprised by pre-existing violations.

**Secrets management:** generate a strong random value for `SECRET_KEY`/Django's `SECRET_KEY` (e.g. `python -c "import secrets; print(secrets.token_urlsafe(50))"`) and any other credential — never hardcode one or leave a placeholder string in code. Write real values only to `.env` (gitignored, step 4); `.env.example` gets the same keys with dummy/empty placeholders so it's safe to commit. Never log settings objects or request bodies that might contain secrets.

### 8. Verify — every item, not just the first

- `docker compose up --build` starts cleanly and the app connects to Postgres
- `/health` and `/ready` both return 200
- `uv run ruff check . && uv run ruff format --check .`
- `uv run mypy .`
- `uv run pytest`
- `uv run pre-commit run --all-files`

Fix and re-run until every check above passes before calling the scaffold done.

If a host port (5432, 8000, ...) is already taken by an unrelated local process, don't "fix" it by editing the committed `docker-compose.yml` ports — that's a host quirk, not a scaffold defect. Use a throwaway remap for the check instead, e.g. `docker compose run --rm -p 18000:8000 app`, and leave the compose file on its standard ports.
