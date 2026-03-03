# Error Handling: Custom Exceptions, Global Handlers, Consistent Responses

## Error Response Envelope

Every error from your API should follow the same shape. Clients should never have to guess the structure.

```python
# Standard error body
{
    "error": {
        "code": "USER_NOT_FOUND",       # Machine-readable, stable across versions
        "message": "User not found",     # Human-readable
        "details": null                  # Optional: validation errors, context
    },
    "request_id": "req_abc123"           # Correlation ID for debugging
}
```

## Custom Exception Hierarchy

Define a base exception and derive domain-specific exceptions from it.

```python
# app/exceptions.py
from fastapi import Request
from fastapi.responses import JSONResponse

class AppException(Exception):
    """Base exception for all application errors."""
    def __init__(
        self,
        status_code: int = 500,
        error_code: str = "INTERNAL_ERROR",
        message: str = "An unexpected error occurred",
        details: dict | list | None = None,
    ):
        self.status_code = status_code
        self.error_code = error_code
        self.message = message
        self.details = details


class NotFoundError(AppException):
    def __init__(self, resource: str = "Resource", identifier: str | int = ""):
        super().__init__(
            status_code=404,
            error_code=f"{resource.upper()}_NOT_FOUND",
            message=f"{resource} not found" + (f": {identifier}" if identifier else ""),
        )


class DuplicateEntityError(AppException):
    def __init__(self, message: str = "Entity already exists"):
        super().__init__(
            status_code=409,
            error_code="DUPLICATE_ENTITY",
            message=message,
        )


class ForbiddenError(AppException):
    def __init__(self, message: str = "You don't have permission to perform this action"):
        super().__init__(
            status_code=403,
            error_code="FORBIDDEN",
            message=message,
        )


class BadRequestError(AppException):
    def __init__(self, message: str, details: dict | list | None = None):
        super().__init__(
            status_code=400,
            error_code="BAD_REQUEST",
            message=message,
            details=details,
        )


class ExternalServiceError(AppException):
    """For failures in downstream services (payment providers, email, etc.)."""
    def __init__(self, service: str, message: str = ""):
        super().__init__(
            status_code=502,
            error_code="EXTERNAL_SERVICE_ERROR",
            message=f"{service} service error" + (f": {message}" if message else ""),
        )
```

## Global Exception Handlers

Register handlers in your app factory so every error goes through the same formatting pipeline.

```python
# app/exceptions.py (continued)
import structlog
from fastapi import Request
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError

logger = structlog.get_logger()

def register_exception_handlers(app):
    @app.exception_handler(AppException)
    async def app_exception_handler(request: Request, exc: AppException):
        return JSONResponse(
            status_code=exc.status_code,
            content={
                "error": {
                    "code": exc.error_code,
                    "message": exc.message,
                    "details": exc.details,
                },
                "request_id": getattr(request.state, "request_id", None),
            },
        )

    @app.exception_handler(RequestValidationError)
    async def validation_exception_handler(request: Request, exc: RequestValidationError):
        """Reformat Pydantic/FastAPI validation errors into our standard envelope."""
        errors = []
        for error in exc.errors():
            errors.append({
                "field": ".".join(str(loc) for loc in error["loc"]),
                "message": error["msg"],
                "type": error["type"],
            })
        return JSONResponse(
            status_code=422,
            content={
                "error": {
                    "code": "VALIDATION_ERROR",
                    "message": "Request validation failed",
                    "details": errors,
                },
                "request_id": getattr(request.state, "request_id", None),
            },
        )

    @app.exception_handler(Exception)
    async def unhandled_exception_handler(request: Request, exc: Exception):
        """Catch-all for unexpected errors. Log the full traceback, return a safe message."""
        request_id = getattr(request.state, "request_id", None)
        logger.error(
            "unhandled_exception",
            request_id=request_id,
            path=request.url.path,
            exc_type=type(exc).__name__,
            exc_msg=str(exc),
            exc_info=True,
        )
        return JSONResponse(
            status_code=500,
            content={
                "error": {
                    "code": "INTERNAL_ERROR",
                    "message": "An unexpected error occurred",
                    "details": None,
                },
                "request_id": request_id,
            },
        )
```

## Using Exceptions in Services

Services raise domain exceptions. Routers don't need try/except — the global handlers catch everything.

```python
# app/users/service.py
class UserService:
    async def get_user(self, user_id: int) -> User:
        user = await self.db.get(User, user_id)
        if not user:
            raise NotFoundError("User", user_id)
        return user

    async def create_user(self, data: UserCreate) -> User:
        existing = await self._get_by_email(data.email)
        if existing:
            raise DuplicateEntityError(f"User with email {data.email} already exists")
        ...
```

```python
# app/users/router.py — clean, no try/except needed
@router.get("/{user_id}", response_model=UserResponse)
async def get_user(user_id: int, db: AsyncSession = Depends(get_db)):
    service = UserService(db)
    return await service.get_user(user_id)
```

## Correlation / Request ID

Inject a request ID in middleware so every log line and error response can be traced:

```python
# app/middleware.py
import uuid
from starlette.middleware.base import BaseHTTPMiddleware

class RequestIdMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        request_id = request.headers.get("X-Request-ID", str(uuid.uuid4()))
        request.state.request_id = request_id

        # Bind to structured logging context
        structlog.contextvars.bind_contextvars(request_id=request_id)

        response = await call_next(request)
        response.headers["X-Request-ID"] = request_id
        return response
```

## Error Codes Registry

Maintain a central registry of error codes so frontend teams can map codes to user-facing messages:

```python
# For documentation / OpenAPI description purposes
ERROR_CODES = {
    "USER_NOT_FOUND": "The requested user does not exist",
    "DUPLICATE_ENTITY": "An entity with the given identifier already exists",
    "VALIDATION_ERROR": "The request body or parameters failed validation",
    "FORBIDDEN": "The authenticated user lacks permission",
    "INTERNAL_ERROR": "An unexpected server error occurred",
    "EXTERNAL_SERVICE_ERROR": "A downstream service failed",
}
```

Include this in your API documentation or as a dedicated endpoint (`GET /api/v1/errors`) so frontend and mobile teams can reference it.
