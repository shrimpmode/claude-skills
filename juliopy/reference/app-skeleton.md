# App skeleton: config, DB, health checks

## Shared: typed settings

Both frameworks read config through one `Settings` class instead of scattered `os.environ` calls:

```python
# app/config.py (FastAPI) or config/settings_env.py (Django)
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")

    database_url: str
    debug: bool = False


settings = Settings()
```

`.env` / `.env.example` supply `DATABASE_URL=postgresql://<user>:<password>@db:5432/<db>` plus `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB` (consumed by the `db` compose service) and `DEBUG`.

## FastAPI

```python
# app/main.py
from fastapi import FastAPI, HTTPException
from sqlalchemy import create_engine, text
from sqlalchemy.orm import sessionmaker

from app.config import settings

app = FastAPI()
engine = create_engine(settings.database_url)
SessionLocal = sessionmaker(bind=engine)


@app.get("/health")
def health() -> dict[str, str]:
    return {"status": "ok"}


@app.get("/ready")
def ready() -> dict[str, str]:
    try:
        with SessionLocal() as session:
            session.execute(text("SELECT 1"))
    except Exception as exc:
        raise HTTPException(status_code=503, detail="database unavailable") from exc
    return {"status": "ready"}
```

Use `asyncpg` + an async engine/session instead if the service is async-heavy end to end — keep `/health` synchronous and dependency-free regardless.

## Django

```python
# config/settings.py
import dj_database_url

from config.settings_env import settings

DATABASES = {"default": dj_database_url.parse(settings.database_url)}
DEBUG = settings.debug
```

```python
# health/views.py
from django.db import connection
from django.http import JsonResponse


def health(request):
    return JsonResponse({"status": "ok"})


def ready(request):
    try:
        with connection.cursor() as cursor:
            cursor.execute("SELECT 1")
    except Exception:
        return JsonResponse({"status": "unavailable"}, status=503)
    return JsonResponse({"status": "ready"})
```

```python
# config/urls.py
from django.urls import path
from health.views import health, ready

urlpatterns = [
    path("health", health),
    path("ready", ready),
    # ... existing routes
]
```

Both frameworks: the Docker `HEALTHCHECK` (see `reference/dockerfile.md`) hits `/health`, not `/ready` — liveness shouldn't fail just because the DB is momentarily slow to accept connections at container start; `depends_on: condition: service_healthy` on `db` already handles that ordering.
