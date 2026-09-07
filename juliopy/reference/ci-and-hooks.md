# CI and pre-commit templates

## .github/workflows/ci.yml

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:<postgres>
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: app_test
        ports: ["5432:5432"]
        options: >-
          --health-cmd "pg_isready -U postgres"
          --health-interval 5s
          --health-timeout 3s
          --health-retries 5
    env:
      DATABASE_URL: postgresql://postgres:postgres@localhost:5432/app_test
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v4
        with:
          version: "<uv>"
      - run: uv sync --frozen
      - run: uv run ruff check .
      - run: uv run ruff format --check .
      - run: uv run mypy .
      - run: uv run pytest
```

Resolve `astral-sh/setup-uv`'s latest tag the same way as everything else (check the action's releases) rather than trusting `v4` above — that's illustrative, not a pin.

## .pre-commit-config.yaml

```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: <ruff-pre-commit-tag>
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: <mirrors-mypy-tag>
    hooks:
      - id: mypy
        additional_dependencies: []  # see below — Django needs far more than django-stubs here

  - repo: https://github.com/gitleaks/gitleaks
    rev: <gitleaks-tag>
    hooks:
      - id: gitleaks
```

Look up each `rev` from the repo's own tags/releases before writing the file — a stale or invented tag will fail `pre-commit install`/`autoupdate`. `gitleaks` blocks a commit that contains an API key, token, or other credential pattern — it's what catches a `SECRET_KEY` or `DATABASE_URL` accidentally pasted into code instead of `.env`.

### The mypy hook's `additional_dependencies` needs runtime deps, not just stubs

The mypy pre-commit hook runs in its own throwaway virtualenv, completely separate from `uv.lock` — `additional_dependencies` is the *only* thing installed into it. mypy still has to *import* your settings module to type-check the project (Django's mypy plugin literally calls `django.apps.populate()`), so every package that import chain touches must be listed here too, pinned to the same version as `pyproject.toml` — not only the `*-stubs` package. For a typical Django scaffold from this skill, that's:

```yaml
        additional_dependencies:
          - "django==<django-version>"
          - "django-stubs==<django-stubs-version>"
          - "pydantic-settings==<pydantic-settings-version>"
          - "dj-database-url==<dj-database-url-version>"
          - "psycopg[binary]==<psycopg-version>"
          - "pytest==<pytest-version>"          # if tests/ is inside mypy's checked paths
          - "pytest-django==<pytest-django-version>"
```

Skipping any of these produces failures that look unrelated to your code — e.g. `django.core.exceptions.ImproperlyConfigured: Error loading psycopg2 or psycopg module` (psycopg missing) or `error: INTERNAL ERROR ... Error constructing plugin instance of NewSemanalDjangoPlugin` (any of the above missing, since Django fails to fully initialize). If you hit either, run the hook's own mypy binary directly with `--show-traceback` to see which import in the chain actually failed, rather than guessing.

After writing both files: `uv run pre-commit install` (registers the git hook) and `uv run pre-commit run --all-files` (verifies it's clean on a fresh scaffold).
