---
name: fastapi-best-practices
description: FastAPI best practices - project structure, routing, services, database, auth, error handling, middleware, testing, deployment, background tasks
user-invocable: false
---

# FastAPI Best Practices

Apply these rules when building or reviewing FastAPI backends.

## Project Structure

Use a modular, domain-driven layout under `app/`. Every domain gets its own package with router, service, schemas, and models co-located.

See [project-structure.md](./project-structure.md) for:
- Directory layout and file placement
- App factory with lifespan
- Router aggregation in `app/api.py`
- Central model registry for Alembic
- Configuration with Pydantic Settings

## Routing & Endpoints

See [routing.md](./routing.md) for:
- Router setup and endpoint patterns
- Response models and status codes
- Pagination (offset and cursor-based)
- API versioning strategies

## Service Layer

Services contain business logic. Routers are thin — validate input, call service, return response.

- Services receive validated data and dependencies, never raw `Request` objects
- Services call `flush()`, not `commit()` — the session dependency handles transaction boundaries
- One session per request via dependency injection

## Database

See [database.md](./database.md) for:
- Async engine setup with `asyncpg` + SQLAlchemy 2.0
- Connection pooling configuration
- Session management via DI
- ORM vs raw SQL guidelines
- Alembic migration patterns

## Authentication & Authorization

See [auth.md](./auth.md) for:
- JWT authentication flow
- OAuth2/OIDC integration
- Role-based access control (RBAC)
- AWS Cognito integration
- Protecting routes with dependencies

## Error Handling

See [errors.md](./errors.md) for:
- Custom exception hierarchy (`AppException` base)
- Global exception handlers
- Consistent error response envelope
- Validation error formatting

## Middleware

See [middleware.md](./middleware.md) for:
- CORS configuration
- Request ID injection
- Request timing/logging
- Registration order (matters for responses)
- `BaseHTTPMiddleware` vs raw ASGI

## Dependency Injection

Use `Depends()` for everything: sessions, auth, pagination, feature flags.

- Domain-specific deps go in `domain/dependencies.py`
- Shared deps go in `app/dependencies.py`
- Avoid chains deeper than 3 levels

## Background Tasks

See [background-tasks.md](./background-tasks.md) for:
- FastAPI `BackgroundTasks` (lightweight only)
- Celery/ARQ integration
- Async worker patterns
- Task retry strategies

## Testing

See [testing.md](./testing.md) for:
- `httpx.AsyncClient` with `ASGITransport`
- Async fixtures and test database setup
- Dependency overrides (not framework mocking)
- Unit vs integration test separation

## Deployment

See [deployment.md](./deployment.md) for:
- Multi-stage Dockerfile
- Gunicorn + UvicornWorker config
- Health check endpoints
- Docker Compose patterns
- Cloud deployment strategies

## Performance

- Use `async def` for I/O, plain `def` for CPU-bound (runs in threadpool)
- Enable `GZipMiddleware` for large payloads
- Connection pool all external services
- Cache with Redis for read-heavy endpoints
- Paginate all list endpoints — never return unbounded collections

## Logging

- Use `structlog` for structured JSON logging in production
- Bind request context (`request_id`, `user_id`) via `contextvars`
- Never log sensitive data (passwords, tokens, PII)

## Quick Reference

| Concern | Location |
|---|---|
| Route definitions | `domain/router.py` |
| Request/response shapes | `domain/schemas.py` |
| Business logic | `domain/service.py` |
| Database models | `domain/models.py` |
| Auth dependencies | `auth/dependencies.py` |
| DB session provider | `app/dependencies.py` |
| Custom exceptions | `app/exceptions.py` |
| Middleware | `app/middleware.py` |
| Env config | `app/config.py` |
| Engine, session, Base | `app/database.py` |
| Model registry (Alembic) | `app/models.py` |
| Router aggregation | `app/api.py` |
