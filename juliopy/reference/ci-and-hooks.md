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
        additional_dependencies: []  # add django-stubs here if using Django
```

Look up each `rev` from the repo's own tags/releases before writing the file — a stale or invented tag will fail `pre-commit install`/`autoupdate`.

After writing both files: `uv run pre-commit install` (registers the git hook) and `uv run pre-commit run --all-files` (verifies it's clean on a fresh scaffold).
