# Routing, Response Models, Pagination (`fastapi-pagination`) & Versioning

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

Use [`fastapi-pagination`](https://github.com/uriyyo/fastapi-pagination) for all paginated endpoints. It handles query params, response schema, and database integration automatically.

### Setup

Call `add_pagination(app)` in your app factory:

```python
# app/main.py
from fastapi_pagination import add_pagination

def create_app() -> FastAPI:
    app = FastAPI(...)
    register_middleware(app)
    register_routers(app)
    register_exception_handlers(app)
    add_pagination(app)  # <-- register pagination
    return app
```

### Page-based pagination (default)

Use `Page[T]` for standard offset/limit pagination with total count:

```python
# app/users/router.py
from fastapi_pagination import Page
from fastapi_pagination.ext.sqlalchemy import paginate
from sqlalchemy import select

@router.get("/", response_model=Page[UserResponse])
async def list_users(
    db: AsyncSession = Depends(get_db),
):
    return await paginate(db, select(User))
```

This auto-adds `page` and `size` query params and returns `items`, `total`, `page`, `size`, `pages`.

### Cursor-based pagination

For large datasets where offset pagination is expensive, use `CursorPage`:

```python
from fastapi_pagination.cursor import CursorPage
from fastapi_pagination.ext.sqlalchemy import paginate
from sqlalchemy import select

@router.get("/", response_model=CursorPage[UserResponse])
async def list_users(
    db: AsyncSession = Depends(get_db),
):
    return await paginate(db, select(User).order_by(User.id))
```

### Combining with filters

```python
@router.get("/", response_model=Page[UserResponse])
async def list_users(
    filters: UserFilter = Depends(get_user_filter),
    db: AsyncSession = Depends(get_db),
):
    query = select(User)
    if filters.role:
        query = query.where(User.role == filters.role)
    if filters.is_active is not None:
        query = query.where(User.is_active == filters.is_active)
    if filters.search:
        query = query.where(User.name.ilike(f"%{filters.search}%"))
    return await paginate(db, query)
```

### Customizing page size

Use `CustomizedPage` to override defaults:

```python
from fastapi_pagination.customization import CustomizedPage, UseParamsFields

# Max 50 items per page, default 20
UserPage = CustomizedPage[
    Page[UserResponse],
    UseParamsFields(size=20),
]

@router.get("/", response_model=UserPage)
async def list_users(db: AsyncSession = Depends(get_db)):
    return await paginate(db, select(User))
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

@router.get("/", response_model=Page[UserResponse])
async def list_users(
    filters: UserFilter = Depends(get_user_filter),
    db: AsyncSession = Depends(get_db),
):
    query = select(User)
    if filters.role:
        query = query.where(User.role == filters.role)
    if filters.search:
        query = query.where(User.name.ilike(f"%{filters.search}%"))
    return await paginate(db, query)
```
