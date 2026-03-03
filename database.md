# Database: Async Engine, Sessions, Pooling, ORM vs Raw SQL, Migrations

## Async Engine Setup

Use SQLAlchemy 2.0 async with `asyncpg` for PostgreSQL.

```python
# app/database.py
from sqlalchemy.ext.asyncio import (
    AsyncSession,
    async_sessionmaker,
    create_async_engine,
)
from app.config import settings

engine = create_async_engine(
    settings.DATABASE_URL,  # postgresql+asyncpg://user:pass@host/db
    pool_size=settings.DATABASE_POOL_SIZE,
    max_overflow=settings.DATABASE_MAX_OVERFLOW,
    pool_pre_ping=True,      # Detect stale connections
    pool_recycle=3600,        # Recycle connections after 1 hour
    echo=settings.DEBUG,      # SQL logging in dev only
)

async_session_factory = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False,   # Prevent lazy-load errors after commit
)

async def init_db():
    """Called during app startup. Verify connectivity."""
    async with engine.begin() as conn:
        await conn.execute(text("SELECT 1"))

async def close_db():
    """Called during app shutdown. Dispose engine and pool."""
    await engine.dispose()
```

### Pool Sizing Guidelines

| Environment | `pool_size` | `max_overflow` | Reasoning |
|---|---|---|---|
| Development | 2 | 3 | Minimal resources |
| Staging | 5 | 10 | Match expected prod ratios |
| Production | 10–20 | 10–20 | Tune based on worker count and DB max_connections |

Formula: `pool_size × num_workers < database max_connections - headroom`. Leave 10-20% headroom for admin connections and migrations.

## Session Dependency

One session per request. Commit on success, rollback on failure.

```python
# app/dependencies.py
from typing import AsyncGenerator
from sqlalchemy.ext.asyncio import AsyncSession
from app.database import async_session_factory

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_factory() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```

This means services should call `flush()` (to get generated IDs, trigger constraints) but never `commit()`. The dependency handles the transaction boundary.

If an endpoint needs multiple independent transactions (rare — usually a code smell), inject a session factory instead:

```python
async def get_session_factory():
    return async_session_factory
```

## ORM Models

Use SQLAlchemy 2.0 `DeclarativeBase` with `Mapped` annotations.

```python
# app/users/models.py
from datetime import datetime
from sqlalchemy import String, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    name: Mapped[str] = mapped_column(String(100))
    hashed_password: Mapped[str] = mapped_column(String(255))
    is_active: Mapped[bool] = mapped_column(default=True)
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(
        server_default=func.now(), onupdate=func.now()
    )
```

Put the shared `Base` in `app/database.py` or `app/models/base.py` — one canonical location that all domain models import from.

## Raw SQL vs ORM

Use the ORM for CRUD and standard queries. Use raw SQL (via `text()`) for:
- Complex reporting queries with CTEs, window functions, lateral joins
- Bulk operations where ORM overhead is measurable
- Database-specific features the ORM doesn't expose cleanly

```python
from sqlalchemy import text

# Raw SQL when complexity warrants it
async def get_usage_report(self, tenant_id: int):
    query = text("""
        WITH daily_usage AS (
            SELECT date_trunc('day', created_at) AS day,
                   COUNT(*) AS count
            FROM events
            WHERE tenant_id = :tenant_id
              AND created_at >= NOW() - INTERVAL '30 days'
            GROUP BY 1
        )
        SELECT day, count,
               SUM(count) OVER (ORDER BY day) AS cumulative
        FROM daily_usage
        ORDER BY day
    """)
    result = await self.db.execute(query, {"tenant_id": tenant_id})
    return result.mappings().all()
```

For bulk inserts, use `insert().values()` instead of adding objects one by one:

```python
from sqlalchemy.dialects.postgresql import insert

stmt = insert(Event).values([
    {"type": "click", "user_id": 1},
    {"type": "view", "user_id": 2},
])
# Upsert pattern
stmt = stmt.on_conflict_do_update(
    index_elements=["id"],
    set_={"type": stmt.excluded.type},
)
await session.execute(stmt)
```

## Alembic Migrations

Initialize Alembic with async support:

```bash
alembic init -t async migrations
```

Configure `migrations/env.py`:

```python
from app.database import Base, engine
target_metadata = Base.metadata

# In run_migrations_online():
async def run_async_migrations():
    async with engine.connect() as connection:
        await connection.run_sync(do_run_migrations)

def do_run_migrations(connection):
    context.configure(connection=connection, target_metadata=target_metadata)
    with context.begin_transaction():
        context.run_migrations()
```

### Migration Best Practices

- Always review auto-generated migrations before applying. Alembic guesses, and it sometimes guesses wrong (especially on renames vs drop+create).
- Name migrations descriptively: `alembic revision --autogenerate -m "add_users_email_index"`.
- Keep migrations reversible — implement `downgrade()` functions.
- For production: run migrations as a separate step before deploying new code (not inside the app startup).
- For data migrations, use a separate migration file — don't mix schema changes with data transforms.

## Using psycopg (v3) Instead of asyncpg

If you prefer `psycopg` (v3) with its native async support:

```python
# Connection string changes to:
# postgresql+psycopg://user:pass@host/db

engine = create_async_engine(
    "postgresql+psycopg://user:pass@host/db",
    pool_size=10,
)
```

`psycopg` v3 has built-in connection pooling via `ConnectionPool` if you want to bypass SQLAlchemy's pool:

```python
from psycopg_pool import AsyncConnectionPool

pool = AsyncConnectionPool(
    conninfo="host=localhost dbname=mydb user=myuser",
    min_size=2,
    max_size=10,
)
```

This is useful for raw-SQL-only projects that don't need the ORM. For most projects, stick with SQLAlchemy's pool — it handles the lifecycle automatically.

## Common Pitfalls

- **N+1 queries:** Use `selectinload()` or `joinedload()` for relationships. Never access lazy-loaded attributes in an async context — it raises `MissingGreenlet`.
- **expire_on_commit:** Set `expire_on_commit=False` on the session factory. Otherwise, accessing attributes after commit triggers a lazy load.
- **Long-running transactions:** Don't hold a session open while making external HTTP calls or waiting on user input. Get your data, release the session, then proceed.
- **Pool exhaustion:** If you see `TimeoutError` from the pool, you're either leaking sessions (not closing them) or need a larger pool. Check that every `get_db` usage properly yields and cleans up.
