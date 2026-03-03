---
name: fastapi-best-practices
description: "Use this skill whenever the user wants to build, scaffold, architect, or debug a Python FastAPI backend. Triggers include: any mention of 'FastAPI', 'fast api', API endpoints, Python backend, REST API with Python, or building a web service in Python. Also use when the user asks about project structure for a Python API, middleware setup, dependency injection patterns, database connection pooling, authentication/authorization for APIs, background tasks, error handling strategies, or deploying a FastAPI app with Docker. Use this skill for any production-level Python API work — even if the user just says 'backend' or 'API' in a Python context. Do NOT use for Flask, Django, or non-Python backends."
---

# FastAPI Production Backend — Best Practices

This skill guides building production-grade FastAPI backends. It covers project structure, routing, services, database access, auth, error handling, middleware, testing, and deployment. The guidance is opinionated toward real-world production systems — not toy examples.

## Project Structure

Use a modular, domain-driven layout under `app/`. Every domain gets its own package with router, service, schemas, and models co-located.

```
project-root/
├── app/
│   ├── __init__.py
│   ├── main.py              # App factory, lifespan, middleware registration
│   ├── config.py             # Pydantic Settings (env-driven config)
│   ├── dependencies.py       # Shared DI providers (db session, current_user, etc.)
│   ├── exceptions.py         # Custom exceptions + global handlers
│   ├── middleware.py          # CORS, logging, request-id, timing
│   ├── database.py           # Engine, session factory, pool config, Base
│   ├── models.py             # Central model registry — imports all domain models
│   ├── api.py                # Router aggregator — mounts all domain routers with versioned prefixes
│   │
│   ├── auth/                 # Auth domain
│   │   ├── router.py
│   │   ├── service.py
│   │   ├── schemas.py
│   │   ├── models.py
│   │   └── dependencies.py   # Domain-specific deps (get_current_user)
│   │
│   ├── users/                # Example domain
│   │   ├── router.py
│   │   ├── service.py
│   │   ├── schemas.py
│   │   └── models.py
│   │
│   └── shared/               # Cross-cutting utilities
│       ├── pagination.py
│       ├── responses.py      # Standardized response envelopes
│       └── validators.py
│
├── migrations/               # Alembic
│   └── versions/
├── tests/
│   ├── conftest.py           # Fixtures: test client, db session, auth overrides
│   ├── test_auth/
│   └── test_users/
├── alembic.ini
├── Dockerfile
├── docker-compose.yml
├── pyproject.toml
└── .env.example
```

**Key principles:**
- One router per domain, aggregated in `app/api.py` with versioned prefixes — `main.py` only imports the single `api_router`. This keeps `main.py` clean even as domains grow.
- Business logic lives in `service.py`, never in routers. Routers are thin — they validate input, call a service, return a response.
- Schemas (Pydantic models) are separate from ORM models. Never return ORM models directly from endpoints.
- Domain-specific dependencies go in the domain's own `dependencies.py`. Shared ones go in `app/dependencies.py`.
- Define `Base = declarative_base()` in `app/database.py`. Create `app/models.py` as a central registry that imports every domain's models (e.g., `from app.auth.models import *`). Point Alembic's `env.py` at this file so `Base.metadata` sees all tables for auto-migration generation.

## App Factory & Lifespan

Use the lifespan context manager for startup/shutdown. This is the modern replacement for `on_event`.

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from app.config import settings
from app.database import init_db, close_db
from app.middleware import register_middleware

@asynccontextmanager
async def lifespan(app: FastAPI):
    await init_db()
    yield
    await close_db()

def create_app() -> FastAPI:
    app = FastAPI(
        title=settings.APP_NAME,
        version=settings.APP_VERSION,
        docs_url="/docs" if settings.ENVIRONMENT != "production" else None,
        lifespan=lifespan,
    )
    register_middleware(app)
    register_routers(app)
    register_exception_handlers(app)
    return app

def register_routers(app: FastAPI):
    from app.api import api_router
    app.include_router(api_router)
```

Disable `/docs` and `/redoc` in production via environment-based settings, not by removing them entirely.

## Router Aggregation (`app/api.py`)

Centralize all domain routers in a single file instead of cluttering `main.py`. This scales cleanly as you add domains.

```python
# app/api.py
from fastapi import APIRouter

api_router = APIRouter(prefix="/api/v1")

def register_domain_routers():
    from app.auth.router import router as auth_router
    from app.users.router import router as users_router

    api_router.include_router(auth_router, prefix="/auth", tags=["auth"])
    api_router.include_router(users_router, prefix="/users", tags=["users"])

register_domain_routers()
```

When you need v2, create a `v2_router` in the same file or a separate `app/api_v2.py` and mount both in `main.py`.

## Central Model Registry (`app/models.py`)

Alembic needs to see all ORM models via `Base.metadata` to generate correct auto-migrations. Since models are distributed across domains, create a central import file.

```python
# app/models.py
# Import all domain models so Base.metadata is fully populated.
# Alembic's env.py should import this file.
from app.auth.models import *  # noqa: F401,F403
from app.users.models import *  # noqa: F401,F403
```

In `migrations/env.py`, point the target metadata here:

```python
from app.database import Base
from app import models  # noqa: F401 — triggers all model imports

target_metadata = Base.metadata
```

## Configuration

Use Pydantic Settings for all configuration. Never use raw `os.getenv` calls scattered through your code.

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=False,
    )

    APP_NAME: str = "MyAPI"
    APP_VERSION: str = "1.0.0"
    ENVIRONMENT: str = "development"  # development | staging | production
    DEBUG: bool = False

    DATABASE_URL: str
    DATABASE_POOL_SIZE: int = 5
    DATABASE_MAX_OVERFLOW: int = 10

    JWT_SECRET_KEY: str
    JWT_ALGORITHM: str = "HS256"
    JWT_EXPIRATION_MINUTES: int = 30

    CORS_ORIGINS: list[str] = ["http://localhost:3000"]

settings = Settings()
```

## Routing & Endpoint Patterns

→ For detailed routing patterns, response models, pagination, and versioning, read `references/routing.md`.

Key rules at a glance:
- Always declare `response_model` on endpoints — it controls serialization and docs.
- Use `status_code` explicitly for non-200 responses (201 for creation, 204 for deletion).
- Keep routers thin: validate, call service, return. No business logic in routers.
- Use `Depends()` for all injectable resources (db, auth, config).

## Service Layer

Services contain business logic. They receive validated data and dependencies, never raw `Request` objects.

```python
class UserService:
    def __init__(self, db: AsyncSession):
        self.db = db

    async def create_user(self, data: UserCreate) -> User:
        # Check uniqueness
        existing = await self.db.execute(
            select(User).where(User.email == data.email)
        )
        if existing.scalar_one_or_none():
            raise DuplicateEntityError("User with this email already exists")

        user = User(**data.model_dump())
        self.db.add(user)
        await self.db.flush()  # Get ID without committing
        return user
```

**Session management:** Services call `flush()`, not `commit()`. The session dependency handles commit/rollback so that multiple service calls in one request share a single transaction.

## Database

→ For detailed database patterns (async engine, session management, connection pooling, raw SQL vs ORM, migrations), read `references/database.md`.

Key rules at a glance:
- Use `asyncpg` + `SQLAlchemy async` for PostgreSQL. Use `psycopg` (v3) as an alternative.
- Configure connection pool size via settings, not hardcoded.
- One session per request via dependency injection. Commit in the dependency's cleanup, not in services.
- Use Alembic for migrations. Never auto-create tables in production.

## Authentication & Authorization

→ For detailed auth patterns (JWT flow, OAuth2/OIDC, role-based access, AWS Cognito integration), read `references/auth.md`.

Key rules at a glance:
- Use OAuth2PasswordBearer for token extraction, custom dependency for validation.
- Separate authentication (who are you?) from authorization (can you do this?).
- Store `get_current_user` as a dependency, compose with role-checking deps.
- Never put auth logic in the service layer — use router-level dependencies.

## Error Handling

→ For detailed error handling patterns (custom exceptions, global handlers, validation error formatting), read `references/errors.md`.

Key rules at a glance:
- Define a base `AppException` with `status_code`, `error_code`, and `message`.
- Register global exception handlers that catch `AppException` subclasses and return a consistent JSON envelope.
- Never let raw 500s escape — catch unexpected exceptions and log them with a correlation ID.
- Validation errors (422) should be reformatted into your standard error envelope.

## Middleware

Register middleware in a dedicated function for clarity. Order matters — middleware executes in reverse registration order for responses.

```python
from fastapi.middleware.cors import CORSMiddleware

def register_middleware(app: FastAPI):
    # CORS — must be first (outermost) for preflight to work
    app.add_middleware(
        CORSMiddleware,
        allow_origins=settings.CORS_ORIGINS,
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )
    # Request ID
    app.add_middleware(RequestIdMiddleware)
    # Logging / timing
    app.add_middleware(TimingMiddleware)
```

For custom middleware, prefer `BaseHTTPMiddleware` for simple cases but know its limitations — it buffers the entire response body. For streaming or WebSocket support, use raw ASGI middleware or Starlette's approach.

## Dependency Injection

FastAPI's DI system via `Depends()` is the backbone of clean architecture. Use it for everything: sessions, auth, pagination params, feature flags.

```python
# app/dependencies.py
async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_factory() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise

# In router
@router.get("/{user_id}")
async def get_user(
    user_id: int,
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user),
):
    service = UserService(db)
    return await service.get_user(user_id)
```

Avoid deeply nested `Depends()` chains (more than 3 levels). If it gets complex, compose at the router level or use a provider class.

## Background Tasks & Workers

→ For detailed patterns on background tasks, Celery/ARQ integration, and async task queues, read `references/background-tasks.md`.

Use FastAPI's built-in `BackgroundTasks` only for fire-and-forget lightweight work (sending emails, logging). For anything that needs retries, monitoring, or takes more than a few seconds, use a proper task queue.

## Testing

→ For detailed testing patterns (fixtures, async tests, mocking, integration vs unit), read `references/testing.md`.

Key rules at a glance:
- Use `httpx.AsyncClient` with `ASGITransport` (not the deprecated TestClient for async apps).
- Override dependencies in tests — don't mock the framework, mock the dependencies.
- Separate unit tests (service logic) from integration tests (full request cycle).
- Use a real test database with transactions rolled back after each test.

## Docker & Deployment

→ For detailed Docker, multi-stage builds, health checks, and cloud deployment patterns, read `references/deployment.md`.

Key rules at a glance:
- Multi-stage Dockerfile: build stage installs deps, runtime stage copies only what's needed.
- Use `gunicorn` with `uvicorn.workers.UvicornWorker` in production, not bare `uvicorn`.
- Health check endpoint at `/health` that verifies DB connectivity.
- Set `--workers` based on CPU cores (2 × cores + 1 as starting point).
- Never bake secrets into images. Use environment variables or a secret manager.

## Performance Considerations

- Use `async def` endpoints when doing I/O (database, HTTP calls, file ops). Use plain `def` for CPU-bound work — FastAPI runs those in a threadpool automatically.
- Enable response compression via `GZipMiddleware` for large payloads.
- Use connection pooling for all external services (database, Redis, HTTP clients).
- Cache aggressively with Redis for read-heavy endpoints. Use `Cache-Control` headers for client-side caching.
- Paginate all list endpoints. Never return unbounded collections.

## Logging

Use structured logging (JSON) in production. `structlog` is the recommended library.

```python
import structlog

logger = structlog.get_logger()

# In middleware or dependency, bind request context
structlog.contextvars.bind_contextvars(request_id=request_id, user_id=user.id)

# Anywhere in the codebase
logger.info("user_created", email=user.email, plan="pro")
```

Never log sensitive data (passwords, tokens, PII). Log at appropriate levels: `debug` for dev, `info` for business events, `warning` for recoverable issues, `error` for failures.

## Quick Reference: What Goes Where

| Concern | Location |
|---|---|
| Route definitions | `domain/router.py` |
| Request/response shapes | `domain/schemas.py` |
| Business logic | `domain/service.py` |
| Database models | `domain/models.py` |
| Auth dependencies | `auth/dependencies.py` |
| DB session provider | `app/dependencies.py` |
| Custom exceptions | `app/exceptions.py` |
| CORS, logging, timing | `app/middleware.py` |
| Env config | `app/config.py` |
| Engine, session factory, Base | `app/database.py` |
| Model registry (Alembic) | `app/models.py` |
| Router aggregation + versioning | `app/api.py` |

## Reference Files

Read these when you need deeper guidance on a specific topic:

| File | When to read |
|---|---|
| `references/routing.md` | Designing endpoints, response models, pagination, API versioning |
| `references/database.md` | Setting up async DB, pooling, session management, raw SQL vs ORM, Alembic |
| `references/auth.md` | JWT auth, OAuth2/OIDC, RBAC, AWS Cognito, protecting routes |
| `references/errors.md` | Custom exceptions, global handlers, consistent error responses |
| `references/testing.md` | Test setup, async fixtures, mocking deps, integration tests |
| `references/deployment.md` | Dockerfile, docker-compose, gunicorn config, health checks, cloud deploy |
| `references/background-tasks.md` | BackgroundTasks, Celery/ARQ, async workers, task patterns |
