# README.md template

Fill in the bracketed parts with the real project name/description and the choices actually
made in steps 1–8 (framework, versions) — don't leave placeholders in the committed file, and
don't pad it with sections that don't apply (drop the Django-only rows from the command table
for a FastAPI project, etc.).

````markdown
# <project name>

<one-line description of what this service does>

## Prerequisites

- Docker and Docker Compose
- [uv](https://docs.astral.sh/uv/) — only needed to run lint/tests/pre-commit outside Docker

## Quick start

```
cp .env.example .env      # fill in real values for local dev
docker compose up --build
```

The app is served at http://localhost:8000 with hot reload — edit code on the host and the
running container picks it up without a rebuild (see "Dev vs. production" below).

Django only — apply migrations once the containers are up:

```
make migrate
```

## Common commands

| Command              | What it does                                          |
| --------------------- | ------------------------------------------------------ |
| `make up`             | Build and start the app + Postgres, with hot reload    |
| `make down`           | Stop and remove containers                             |
| `make logs`           | Tail the app container's logs                          |
| `make shell`          | Open a Django shell inside the app container           |
| `make migrate`        | Apply database migrations                               |
| `make makemigrations` | Generate new migrations from model changes              |
| `make test`           | Run the test suite (`pytest`)                           |
| `make lint`           | Run ruff + mypy checks                                  |
| `make fmt`            | Auto-fix lint/format issues                              |
| `make precommit`      | Run all pre-commit hooks against the whole repo          |

(No Makefile target replaces `git commit` — pre-commit hooks run automatically on commit once
`uv run pre-commit install` has been run, which the scaffold already did.)

## Environment variables

See `.env.example` for the full list and what each one is for. `.env` itself is gitignored and
holds real local secrets — never commit it, never paste a real value into `.env.example`.

## Dev vs. production

`docker compose up` (no flags) merges `docker-compose.override.yml` on top of
`docker-compose.yml` automatically — that's what gives local dev the `dev` build target, hot
reload, and `DEBUG=True`. A production build/deploy should invoke Compose (or just `docker
build`) against `docker-compose.yml` alone, explicitly, so the override never applies:

```
docker compose -f docker-compose.yml up --build
```

That builds the `production` target: no dev tools, no bind mount, no autoreload, runs as a
non-root user behind gunicorn.

## Health checks

- `GET /health` — liveness only, no I/O. What the Docker `HEALTHCHECK` polls.
- `GET /ready` — checks the database connection; use this for a load balancer/orchestrator
  readiness probe instead of `/health`.

## Project structure

<brief tree + one line per top-level item — e.g. for Django:>

```
config/          # settings, urls, wsgi/asgi entry points
<app>/           # Django app(s)
tests/           # pytest test suite
docker-compose.yml            # base services: app (production target) + Postgres
docker-compose.override.yml   # local dev overrides, applied automatically
Dockerfile                    # dev / builder / production build stages
```
````
