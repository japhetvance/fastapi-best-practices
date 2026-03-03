# Routing, Response Models, Pagination & Versioning

## Router Setup

Each domain gets a single `APIRouter` instance. Prefix and tags are applied when mounting in `app/api.py`, not in the router itself.

```python
# app/users/router.py
from fastapi import APIRouter, Depends, status
from sqlalchemy.ext.asyncio import AsyncSession
from app.dependencies import get_db
from app.auth.dependencies import get_current_user
from app.users.models import User
from .service import UserService
from .schemas import UserCreate, UserResponse

router = APIRouter()

@router.post(
    "/",
    response_model=UserResponse,
    status_code=status.HTTP_201_CREATED,
    summary="Create a new user",
)
async def create_user(
    data: UserCreate,
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user),
):
    service = UserService(db)
    user = await service.create_user(data)
    return user
```

## Response Models & Envelopes

Always use explicit Pydantic response models. Never return raw dicts or ORM objects.

```python
# app/users/schemas.py
from pydantic import BaseModel, ConfigDict, EmailStr
from datetime import datetime

# --- Input schemas ---
class UserCreate(BaseModel):
    email: EmailStr
    name: str
    role: str = "member"

class UserUpdate(BaseModel):
    name: str | None = None
    role: str | None = None

# --- Output schemas ---
class UserResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    email: str
    name: str
    role: str
    created_at: datetime

# --- Standardized envelope ---
class DataResponse[T](BaseModel):
    data: T
    meta: dict | None = None
```

**Naming convention:** `XCreate`, `XUpdate` for inputs. `XResponse` for outputs. `XFilter` for query params.

### Partial Updates (PATCH)

For PATCH endpoints, use `model.model_dump(exclude_unset=True)` to distinguish between "not provided" and "set to None":

```python
@router.patch("/{user_id}", response_model=UserResponse)
async def update_user(
    user_id: int,
    data: UserUpdate,
    db: AsyncSession = Depends(get_db),
):
    service = UserService(db)
    updates = data.model_dump(exclude_unset=True)
    return await service.update_user(user_id, updates)
```

## Pagination

Use cursor-based pagination for production. Offset pagination is acceptable for internal/admin tools.

```python
# app/shared/pagination.py
from pydantic import BaseModel, Field

class PaginationParams(BaseModel):
    limit: int = Field(default=20, ge=1, le=100)
    cursor: str | None = None

class PaginatedResponse[T](BaseModel):
    items: list[T]
    next_cursor: str | None = None
    has_more: bool

# As a dependency
from fastapi import Query

def get_pagination(
    limit: int = Query(default=20, ge=1, le=100),
    cursor: str | None = Query(default=None),
) -> PaginationParams:
    return PaginationParams(limit=limit, cursor=cursor)
```

**Cursor-based pagination in the service layer:**

```python
async def list_users(self, params: PaginationParams) -> PaginatedResponse[User]:
    query = select(User).order_by(User.id)

    if params.cursor:
        cursor_id = decode_cursor(params.cursor)
        query = query.where(User.id > cursor_id)

    query = query.limit(params.limit + 1)  # Fetch one extra to check has_more
    result = await self.db.execute(query)
    rows = result.scalars().all()

    has_more = len(rows) > params.limit
    items = rows[:params.limit]

    return PaginatedResponse(
        items=items,
        has_more=has_more,
        next_cursor=encode_cursor(items[-1].id) if has_more else None,
    )
```

## API Versioning

Use URL-prefix versioning (`/api/v1/`, `/api/v2/`). It's explicit and works with every client and proxy.

```python
# app/api.py
v1 = APIRouter(prefix="/api/v1")
v1.include_router(auth_router, prefix="/auth", tags=["auth"])
v1.include_router(users_router, prefix="/users", tags=["users"])

v2 = APIRouter(prefix="/api/v2")
v2.include_router(users_v2_router, prefix="/users", tags=["users-v2"])

# Export a single router for main.py to mount
api_router = APIRouter()
api_router.include_router(v1)
api_router.include_router(v2)
```

When introducing v2:
- Keep v1 working. Deprecate with response headers (`Deprecation: true`, `Sunset: <date>`).
- v2 routers can reuse v1 services if the logic hasn't changed — just swap schemas.
- Remove v1 only after a documented sunset period.

## File Uploads

Use `UploadFile` for file handling. Validate file type and size early.

```python
from fastapi import UploadFile, File, HTTPException

MAX_FILE_SIZE = 10 * 1024 * 1024  # 10MB
ALLOWED_TYPES = {"application/pdf", "image/png", "image/jpeg"}

@router.post("/upload")
async def upload_file(
    file: UploadFile = File(...),
):
    if file.content_type not in ALLOWED_TYPES:
        raise HTTPException(400, f"File type {file.content_type} not allowed")

    contents = await file.read()
    if len(contents) > MAX_FILE_SIZE:
        raise HTTPException(400, "File exceeds 10MB limit")

    # Process file...
    return {"filename": file.filename, "size": len(contents)}
```

For large files, stream instead of reading into memory:

```python
import aiofiles

async with aiofiles.open(f"/tmp/{file.filename}", "wb") as out:
    while chunk := await file.read(8192):
        await out.write(chunk)
```

## Query Parameter Patterns

Use Pydantic models with `Query()` for complex filters:

```python
from fastapi import Depends, Query

class UserFilter(BaseModel):
    role: str | None = None
    is_active: bool | None = None
    search: str | None = None
    created_after: datetime | None = None

def get_user_filter(
    role: str | None = Query(None),
    is_active: bool | None = Query(None),
    search: str | None = Query(None, min_length=1, max_length=100),
    created_after: datetime | None = Query(None),
) -> UserFilter:
    return UserFilter(
        role=role, is_active=is_active,
        search=search, created_after=created_after,
    )

@router.get("/", response_model=PaginatedResponse[UserResponse])
async def list_users(
    filters: UserFilter = Depends(get_user_filter),
    pagination: PaginationParams = Depends(get_pagination),
    db: AsyncSession = Depends(get_db),
):
    service = UserService(db)
    return await service.list_users(filters, pagination)
```
