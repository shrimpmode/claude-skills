# Dev environment: hot reload, docker-compose.override.yml, Makefile

This is what makes local development pleasant on top of the production-shaped scaffold from
`reference/dockerfile.md`: edit a file on the host, see the running container pick it up
without a manual rebuild.

## docker-compose.override.yml

Docker Compose automatically reads and merges `docker-compose.override.yml` on top of
`docker-compose.yml` for a bare `docker compose up` — no flag needed. CI and any real deploy
should invoke `docker compose -f docker-compose.yml ...` explicitly so the override never
applies there; that's also why `reference/dockerfile.md`'s base compose file targets
`production` on its own.

```yaml
services:
  app:
    image: <project>-app:dev
    build:
      target: dev
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
    # Django, use instead:
    # command: python manage.py runserver 0.0.0.0:8000
    environment:
      DEBUG: "True"
    volumes:
      - .:/app
      - /app/.venv
```

Three things that are easy to get wrong here:

- **`image: <project>-app:dev`** (substitute the real project name) is not optional. Compose
  otherwise builds every target for a given service to the *same* default image name/tag
  (`<project>-<service>`), so building `production` (e.g. from CI, or a plain `docker compose
  -f docker-compose.yml build`) and then running `docker compose run app ...` without `--build`
  silently reuses whichever image was built last — dev tooling like `uv`/`pytest` then appears
  to be "missing" from a container that's actually just running the production image under a
  dev config. Giving the dev build its own tag makes the two images impossible to confuse, and
  Compose still rebuilds it automatically for `up`/`run` when the Dockerfile or context changes.
- **`target: dev`** points the build at the `dev` stage from `reference/dockerfile.md` (dev
  tools installed, no non-root user, no `HEALTHCHECK`) instead of `production`.
- **The `/app/.venv` line is not redundant with `.:/app`.** Bind-mounting the whole project
  directory (`.:/app`) shadows everything already inside the image's `/app`, including the
  `.venv` that `uv sync` built into the image during `docker build` — without a second mount
  entry for `/app/.venv` specifically, the container would boot with an empty or host-mismatched
  virtualenv and fail on import. A `volumes:` entry with no host source (just a container path)
  creates an anonymous volume that Compose layers on top of the bind mount for that one
  subpath, so the image's own `.venv` wins there while everything else stays bind-mounted from
  the host. This is the same trick as excluding `node_modules` from a bind mount in a JS
  project — same root cause, same fix.
- `environment: DEBUG: "True"` overrides whatever `.env` (loaded via the base file's
  `env_file:`) says for local dev, without having to keep two different `.env` files in sync.
  `environment:` values win over `env_file:` values for the same key when Compose merges them.

Both frameworks' built-in dev servers (`runserver`'s StatReloader, `uvicorn --reload`)
poll/watch the filesystem and work fine against a bind mount without extra configuration —
don't add polling env vars or file-watcher tuning unless reload is actually observed to be
flaky in a specific environment.

## Makefile

Optional but recommended — gives a consistent, memorable set of commands regardless of
framework, instead of everyone remembering their own `docker compose exec` incantations. Adjust
the Django-only targets (`migrate`, `makemigrations`, `shell`) out if scaffolding FastAPI —
FastAPI has no built-in migration tool; if the project needs one, that's a separate decision
(e.g. Alembic) outside this skill's scope.

```makefile
.PHONY: up down build logs shell migrate makemigrations test lint fmt precommit

up:
	docker compose up --build

down:
	docker compose down

build:
	docker compose build

logs:
	docker compose logs -f app

shell:
	docker compose exec app python manage.py shell

migrate:
	docker compose exec app python manage.py migrate

makemigrations:
	docker compose exec app python manage.py makemigrations

test:
	uv run pytest

lint:
	uv run ruff check . && uv run ruff format --check . && uv run mypy .

fmt:
	uv run ruff check --fix . && uv run ruff format .

precommit:
	uv run pre-commit run --all-files
```

`test`/`lint`/`fmt`/`precommit` run via `uv run` on the host, not inside the container — that
matches what CI does (`reference/ci-and-hooks.md`) and avoids needing the dev container up just
to lint. `migrate`/`makemigrations`/`shell` do need the container running (`make up` first)
since they touch the app's live DB connection.

Migrations are **not** run automatically on container start — that's deliberate, so a schema
change is always something a developer explicitly chose to apply, not a side effect of
restarting the stack. Say this plainly in the README (`reference/readme-template.md`) so it
isn't mistaken for an oversight.
