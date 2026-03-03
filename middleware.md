# Middleware: CORS, Request ID, Timing, Custom Middleware

## Registration

Register middleware in a dedicated function for clarity. Order matters — middleware executes in reverse registration order for responses.

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from app.config import settings

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

## Request ID Middleware

Inject a unique request ID into every request for tracing across logs.

```python
import uuid
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request

class RequestIdMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        request_id = request.headers.get("X-Request-ID", str(uuid.uuid4()))
        request.state.request_id = request_id
        response = await call_next(request)
        response.headers["X-Request-ID"] = request_id
        return response
```

## Timing Middleware

Log request duration for performance monitoring.

```python
import time
import structlog
from starlette.middleware.base import BaseHTTPMiddleware

logger = structlog.get_logger()

class TimingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        start = time.perf_counter()
        response = await call_next(request)
        duration_ms = (time.perf_counter() - start) * 1000
        logger.info(
            "request_completed",
            method=request.method,
            path=request.url.path,
            status=response.status_code,
            duration_ms=round(duration_ms, 2),
        )
        return response
```

## BaseHTTPMiddleware vs Raw ASGI

- **`BaseHTTPMiddleware`**: Simple to write, good for most cases. Limitation: it buffers the entire response body, so avoid for streaming responses or WebSocket endpoints.
- **Raw ASGI middleware**: Use when you need streaming support, WebSocket handling, or maximum performance. More boilerplate but no buffering.

```python
# Raw ASGI middleware example
class RawTimingMiddleware:
    def __init__(self, app):
        self.app = app

    async def __call__(self, scope, receive, send):
        if scope["type"] != "http":
            await self.app(scope, receive, send)
            return

        start = time.perf_counter()
        await self.app(scope, receive, send)
        duration_ms = (time.perf_counter() - start) * 1000
        logger.info("request_completed", duration_ms=round(duration_ms, 2))
```

## GZip Compression

Enable for large JSON payloads. Set a minimum size threshold to avoid compressing tiny responses.

```python
from fastapi.middleware.gzip import GZipMiddleware

app.add_middleware(GZipMiddleware, minimum_size=1000)  # bytes
```

## Trusted Host / HTTPS Redirect

For production behind a reverse proxy:

```python
from starlette.middleware.trustedhost import TrustedHostMiddleware
from starlette.middleware.httpsredirect import HTTPSRedirectMiddleware

if settings.ENVIRONMENT == "production":
    app.add_middleware(TrustedHostMiddleware, allowed_hosts=settings.ALLOWED_HOSTS)
    app.add_middleware(HTTPSRedirectMiddleware)
```
