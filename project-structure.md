# Project Structure, App Factory & Configuration

## Directory Layout

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
- One router per domain, aggregated in `app/api.py` with versioned prefixes — `main.py` only imports the single `api_router`
- Business logic lives in `service.py`, never in routers. Routers are thin — validate input, call service, return response
- Schemas (Pydantic models) are separate from ORM models. Never return ORM models directly from endpoints
- Domain-specific dependencies go in the domain's own `dependencies.py`. Shared ones go in `app/dependencies.py`
- Define `Base` using SQLAlchemy 2.0 `DeclarativeBase` in `app/database.py`. Create `app/models.py` as a central registry that imports every domain's models

## App Factory & Lifespan

Use the lifespan context manager for startup/shutdown. This is the modern replacement for `on_event`.

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from app.config import settings
from app.database import init_db, close_db
from app.exceptions import register_exception_handlers
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
