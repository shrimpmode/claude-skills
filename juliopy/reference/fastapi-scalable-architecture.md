# FastAPI: package-by-feature layered architecture (scalable variant)

Use this instead of `reference/app-skeleton.md`'s single-module FastAPI skeleton once the
service is expected to grow past a handful of endpoints, has more than one resource/domain, or
the user asks for something "scalable"/"production-grade." For a genuine one-endpoint
prototype, the flat skeleton is still the right call — don't impose this structure on something
too small to need it (SKILL.md step 1 decides which).

## Why this shape

The pattern that holds up in production FastAPI codebases at scale is **package by feature, not
by layer type**. A single project-wide `app/routers/`, `app/models/`, `app/schemas/` works for a
handful of endpoints, then becomes a bottleneck: every unrelated feature touches the same three
folders, nothing is independently testable or extractable, and diffs collide constantly.
Packaging by feature (module = bounded context) keeps each domain self-contained — router,
schema, model, service, repository together — so a module can be read, tested, or eventually
pulled out into its own service without touching the rest of the app.

Within each module, keep four layers strictly separated:

- **router** — HTTP only: parses the request, calls the service, returns the response. No
  business logic, no direct DB/ORM access.
- **service** — business logic and orchestration. Talks to the repository, never the ORM
  directly. Raises domain exceptions, not `HTTPException` — the router layer (or a centralized
  handler) is what translates those to HTTP.
- **repository** — the only place that touches SQLAlchemy. Returns domain/ORM objects; never
  leaks a raw query or session into the service layer.
- **schemas** — pydantic request/response models. Never reuse ORM models as API contracts — a
  column rename or an added relationship shouldn't silently change the wire format.

This separation is what makes the architecture "scalable" in the load-bearing sense: the router
layer is replaceable (add a CLI or gRPC entrypoint later without touching business logic), the
repository is swappable in tests (inject a fake), and business rules live in exactly one place
instead of scattered across handlers.

## Directory layout

```
app/
  main.py                  # app factory, mounts routers, lifespan, exception handlers
  core/
    config.py              # Settings (pydantic-settings)
    db.py                  # async engine, sessionmaker, Base
  api/
    deps.py                # shared dependencies: DbSession, pagination, current user
    v1/
      router.py            # aggregates each module's router under /api/v1
  modules/
    users/
      router.py
      schemas.py
      models.py
      service.py
      repository.py
      exceptions.py
    items/
      router.py
      schemas.py
      models.py
      service.py
      repository.py
      exceptions.py
alembic/
  env.py
  versions/
tests/
  modules/
    users/
      test_router.py
      test_service.py
```

Add a new top-level package under `modules/` per bounded context. Resist a project-wide
`utils.py` grab-bag — if logic is genuinely shared across modules, it belongs in `core/` and
should earn its place there, not accumulate by default.

## core/config.py — extend the shared Settings

Builds on the shared `Settings` pattern in `reference/app-skeleton.md`, adding pool tuning:

```python
class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")

    database_url: str
    secret_key: str
    debug: bool = False
    db_pool_size: int = 5
    db_max_overflow: int = 10


settings = Settings()  # type: ignore[call-arg]
```

`db_pool_size`/`db_max_overflow` matter once there's more than one uvicorn worker or replica:
each worker process opens its own pool, so `workers × replicas × (pool_size + max_overflow)` is
the real number of Postgres connections the app can open — check that against Postgres's
`max_connections` before scaling replicas up, not after a "too many clients already" outage.

## core/db.py — async engine and session

```python
from collections.abc import AsyncGenerator

from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine
from sqlalchemy.orm import DeclarativeBase

from app.core.config import settings


class Base(DeclarativeBase):
    pass


engine = create_async_engine(
    settings.database_url,
    pool_size=settings.db_pool_size,
    max_overflow=settings.db_max_overflow,
    pool_pre_ping=True,
)
AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False)


async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionLocal() as session:
        yield session
```

`database_url` needs the `+asyncpg` driver scheme (`postgresql+asyncpg://...`) for this engine
— the plain `postgresql://` scheme from `reference/app-skeleton.md`'s sync example won't work
here. `pool_pre_ping=True` catches a connection Postgres already closed (idle timeout, restart)
before it's handed to a request, trading one cheap `SELECT 1` for avoiding an intermittent
`ConnectionResetError` under load — worth it at any real traffic level.

`expire_on_commit=False` avoids `MissingGreenlet` errors when a response schema reads ORM
attributes after the session's transaction has already committed inside a repository call — a
common surprise the first time async SQLAlchemy meets FastAPI's per-request session lifecycle.

## api/deps.py

```python
from typing import Annotated

from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession

from app.core.db import get_db

DbSession = Annotated[AsyncSession, Depends(get_db)]
```

The `Annotated` alias (PEP 593, supported since FastAPI 0.95+) goes in every module's router
signature (`async def get_user(user_id: int, db: DbSession)`) instead of repeating
`db: AsyncSession = Depends(get_db)` everywhere — one definition, reused across modules, and the
type checker sees the real type rather than `Depends`'s.

## A module end to end (users)

```python
# app/modules/users/models.py
from sqlalchemy.orm import Mapped, mapped_column

from app.core.db import Base


class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(unique=True, index=True)
```

```python
# app/modules/users/schemas.py
from pydantic import BaseModel, ConfigDict


class UserRead(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    email: str


class UserCreate(BaseModel):
    email: str
```

```python
# app/modules/users/exceptions.py
class UserNotFoundError(Exception):
    def __init__(self, user_id: int) -> None:
        self.user_id = user_id
```

```python
# app/modules/users/repository.py
from sqlalchemy.ext.asyncio import AsyncSession

from app.modules.users.models import User


class UserRepository:
    def __init__(self, db: AsyncSession) -> None:
        self._db = db

    async def get(self, user_id: int) -> User | None:
        return await self._db.get(User, user_id)

    async def create(self, email: str) -> User:
        user = User(email=email)
        self._db.add(user)
        await self._db.commit()
        await self._db.refresh(user)
        return user
```

```python
# app/modules/users/service.py
from app.modules.users.exceptions import UserNotFoundError
from app.modules.users.models import User
from app.modules.users.repository import UserRepository


class UserService:
    def __init__(self, repo: UserRepository) -> None:
        self._repo = repo

    async def get(self, user_id: int) -> User:
        user = await self._repo.get(user_id)
        if user is None:
            raise UserNotFoundError(user_id)
        return user

    async def create(self, email: str) -> User:
        return await self._repo.create(email)
```

```python
# app/modules/users/router.py
from fastapi import APIRouter

from app.api.deps import DbSession
from app.modules.users.models import User
from app.modules.users.repository import UserRepository
from app.modules.users.schemas import UserCreate, UserRead
from app.modules.users.service import UserService

router = APIRouter(prefix="/users", tags=["users"])


def get_service(db: DbSession) -> UserService:
    return UserService(UserRepository(db))


@router.get("/{user_id}", response_model=UserRead)
async def get_user(user_id: int, db: DbSession) -> User:
    return await get_service(db).get(user_id)


@router.post("/", response_model=UserRead, status_code=201)
async def create_user(body: UserCreate, db: DbSession) -> User:
    return await get_service(db).create(body.email)
```

The router never imports SQLAlchemy directly and never touches `UserRepository`'s internals —
swapping how users are persisted later means changing `repository.py` alone.

## app/api/v1/router.py

```python
from fastapi import APIRouter

from app.modules.items.router import router as items_router
from app.modules.users.router import router as users_router

api_router = APIRouter()
api_router.include_router(users_router)
api_router.include_router(items_router)
```

Versioning the API from day one (`/api/v1/...`) costs nothing now and avoids a breaking-change
migration later when a `v2` actually becomes necessary — add `app/api/v2/router.py` alongside,
not instead of, `v1` when that day comes.

## app/main.py — app factory, lifespan, centralized exception handling

```python
from contextlib import asynccontextmanager
from typing import AsyncIterator

from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from sqlalchemy import text

from app.api.deps import DbSession
from app.api.v1.router import api_router
from app.modules.users.exceptions import UserNotFoundError


@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncIterator[None]:
    yield  # startup goes before yield, shutdown after — nothing needed yet


def create_app() -> FastAPI:
    app = FastAPI(lifespan=lifespan)
    app.include_router(api_router, prefix="/api/v1")

    @app.exception_handler(UserNotFoundError)
    async def user_not_found_handler(request: Request, exc: UserNotFoundError) -> JSONResponse:
        return JSONResponse(status_code=404, content={"detail": f"user {exc.user_id} not found"})

    @app.get("/health")
    def health() -> dict[str, str]:
        return {"status": "ok"}

    @app.get("/ready")
    async def ready(db: DbSession) -> dict[str, str]:
        await db.execute(text("SELECT 1"))
        return {"status": "ready"}

    return app


app = create_app()
```

Mapping domain exceptions to HTTP responses once, here, replaces a `try/except` block in every
router that could raise `UserNotFoundError` — add one `@app.exception_handler(...)` per domain
exception type as modules grow, rather than importing `HTTPException` into service code.

`create_app()` as a factory — not a bare module-level `app = FastAPI()` — is what lets tests
build a fresh app instance with dependency overrides (see Testing below), and is where
`lifespan` startup/shutdown code goes; FastAPI's `on_event("startup")` is deprecated in favor of
`lifespan`.

## Migrations: Alembic (async)

FastAPI has no built-in migration tool. For anything beyond a throwaway prototype,
`Base.metadata.create_all()` doesn't survive a second developer or a production deploy — add
Alembic:

```
uv add alembic
uv run alembic init -t async alembic
```

The `-t async` template generates an `alembic/env.py` already wired for an async engine — point
it at your `Base` and `settings.database_url`, and import every module's `models.py` so
autogenerate can see their tables:

```python
# alembic/env.py (relevant edits)
from app.core.config import settings
from app.core.db import Base
from app.modules.items.models import Item  # noqa: F401
from app.modules.users.models import User  # noqa: F401

config.set_main_option("sqlalchemy.url", settings.database_url)
target_metadata = Base.metadata
```

Every module's models must be imported somewhere `env.py` reaches, or
`alembic revision --autogenerate` silently produces an empty migration for a new module's
tables — this is the single most common Alembic mistake and it fails quietly, not loudly.

Makefile additions (alongside the targets in `reference/dev-environment.md`; this supersedes
that file's note that FastAPI has no migration story):

```makefile
migrate:
	docker compose exec app alembic upgrade head

makemigrations:
	docker compose exec app alembic revision --autogenerate -m "$(m)"
```

`make makemigrations m="add users table"` — the `m=` argument is required, Alembic has no
default message. Migrations are not run automatically on container start, same rule as the
Django path — a schema change is always an explicit developer action.

## Testing: dependency overrides, not a real network call

```python
# tests/conftest.py
import pytest
from httpx import ASGITransport, AsyncClient

from app.core.db import get_db
from app.main import create_app


@pytest.fixture
async def client(db_session):
    app = create_app()
    app.dependency_overrides[get_db] = lambda: db_session
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as ac:
        yield ac
```

`app.dependency_overrides` swaps `get_db` for a fixture-provided session (pointed at a test
database, or a per-test transaction rolled back afterward) without touching any application code
— this is the payoff of routing every DB access through the one `get_db` dependency instead of
importing a module-level session directly. Add `httpx` as a dev dependency
(`uv add --dev httpx`) — it isn't pulled in by plain `fastapi`.

## Production concurrency: workers, not just replicas

The base skeleton's `CMD ["uvicorn", "app.main:app", ...]` (`reference/dockerfile.md`) runs a
single process. One async process handles many concurrent I/O-bound requests fine, but uses only
one CPU core — beyond a small deployment, run multiple worker processes per container and scale
replicas on top of that, not instead of it:

```dockerfile
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

Pick the worker count from the container's actual CPU allocation (`2 × cores + 1` is the common
starting heuristic, same as gunicorn's), not a fixed number copied between projects — and
remember each worker opens its own DB pool (`core/config.py` above), so
`workers × replicas × (pool_size + max_overflow)` is the number to check against Postgres's
`max_connections`, not `pool_size` alone.
