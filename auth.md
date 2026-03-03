# Authentication & Authorization

## JWT Authentication Flow

### Token Service

```python
# app/auth/service.py
from datetime import datetime, timedelta, timezone
from jose import JWTError, jwt
from passlib.context import CryptContext
from app.config import settings

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def hash_password(password: str) -> str:
    return pwd_context.hash(password)

def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)

def create_access_token(subject: str, extra_claims: dict | None = None) -> str:
    now = datetime.now(timezone.utc)
    claims = {
        "sub": subject,
        "iat": now,
        "exp": now + timedelta(minutes=settings.JWT_EXPIRATION_MINUTES),
        "type": "access",
    }
    if extra_claims:
        claims.update(extra_claims)
    return jwt.encode(claims, settings.JWT_SECRET_KEY, algorithm=settings.JWT_ALGORITHM)

def create_refresh_token(subject: str) -> str:
    now = datetime.now(timezone.utc)
    claims = {
        "sub": subject,
        "iat": now,
        "exp": now + timedelta(days=7),
        "type": "refresh",
    }
    return jwt.encode(claims, settings.JWT_SECRET_KEY, algorithm=settings.JWT_ALGORITHM)

def decode_token(token: str) -> dict:
    """Raises JWTError on invalid/expired token."""
    return jwt.decode(
        token, settings.JWT_SECRET_KEY, algorithms=[settings.JWT_ALGORITHM]
    )
```

### Auth Dependencies

```python
# app/auth/dependencies.py
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from jose import JWTError
from sqlalchemy.ext.asyncio import AsyncSession
from app.dependencies import get_db
from .service import decode_token

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/v1/auth/login")

async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db),
) -> User:
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Invalid or expired token",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        payload = decode_token(token)
        user_id = payload.get("sub")
        token_type = payload.get("type")
        if user_id is None or token_type != "access":
            raise credentials_exception
    except JWTError:
        raise credentials_exception

    user = await db.get(User, int(user_id))
    if user is None or not user.is_active:
        raise credentials_exception
    return user
```

### Auth Router

```python
# app/auth/router.py
from fastapi import APIRouter, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordRequestForm
from .service import verify_password, create_access_token, create_refresh_token
from .schemas import TokenResponse

router = APIRouter()

@router.post("/login", response_model=TokenResponse)
async def login(
    form: OAuth2PasswordRequestForm = Depends(),
    db: AsyncSession = Depends(get_db),
):
    user = await get_user_by_email(db, form.username)
    if not user or not verify_password(form.password, user.hashed_password):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect email or password",
        )

    return TokenResponse(
        access_token=create_access_token(str(user.id)),
        refresh_token=create_refresh_token(str(user.id)),
        token_type="bearer",
    )
```

## Role-Based Access Control (RBAC)

Compose role checks as reusable dependencies:

```python
# app/auth/dependencies.py
from functools import wraps

def require_role(*allowed_roles: str):
    """Dependency factory for role-based access."""
    async def role_checker(
        current_user: User = Depends(get_current_user),
    ) -> User:
        if current_user.role not in allowed_roles:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail="Insufficient permissions",
            )
        return current_user
    return role_checker

# Usage in router
@router.delete("/{user_id}", dependencies=[Depends(require_role("admin"))])
async def delete_user(user_id: int, db: AsyncSession = Depends(get_db)):
    ...

# Or inject the user when you need it in the handler
@router.get("/admin/dashboard")
async def admin_dashboard(
    current_user: User = Depends(require_role("admin", "manager")),
):
    ...
```

For fine-grained permissions (beyond roles), use a permission enum and check against a user-permissions join table:

```python
class Permission(str, Enum):
    READ_USERS = "read:users"
    WRITE_USERS = "write:users"
    DELETE_USERS = "delete:users"
    MANAGE_BILLING = "manage:billing"

def require_permission(*perms: Permission):
    async def checker(current_user: User = Depends(get_current_user)):
        user_perms = {p.name for p in current_user.permissions}
        if not all(p.value in user_perms for p in perms):
            raise HTTPException(403, "Missing required permissions")
        return current_user
    return checker
```

## External Identity Provider (OAuth2/OIDC)

When using an external IdP (AWS Cognito, Auth0, Okta, etc.), FastAPI validates the JWT issued by the IdP — it doesn't issue tokens itself.

```python
# app/auth/dependencies.py
from jose import jwt, jwk
import httpx

JWKS_URL = settings.OIDC_JWKS_URL  # e.g. https://cognito-idp.{region}.amazonaws.com/{pool_id}/.well-known/jwks.json
ISSUER = settings.OIDC_ISSUER
AUDIENCE = settings.OIDC_CLIENT_ID

_jwks_cache: dict | None = None

async def get_jwks() -> dict:
    global _jwks_cache
    if _jwks_cache is None:
        async with httpx.AsyncClient() as client:
            resp = await client.get(JWKS_URL)
            _jwks_cache = resp.json()
    return _jwks_cache

async def get_current_user_oidc(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db),
) -> User:
    try:
        jwks = await get_jwks()
        header = jwt.get_unverified_header(token)
        key = next(k for k in jwks["keys"] if k["kid"] == header["kid"])
        payload = jwt.decode(
            token,
            key,
            algorithms=["RS256"],
            audience=AUDIENCE,
            issuer=ISSUER,
        )
    except Exception:
        raise HTTPException(401, "Invalid token")

    sub = payload["sub"]
    user = await get_or_create_user_from_sub(db, sub, payload)
    return user
```

### AWS Cognito Specifics

- Cognito issues two tokens: `id_token` (user attributes) and `access_token` (API authorization). Use the `access_token` for API auth.
- The `sub` claim is the stable user identifier across token refreshes.
- Cache the JWKS response — keys rotate infrequently (weeks/months). Invalidate on `kid` mismatch.
- Cognito's `access_token` audience claim is the app client ID.

## Security Reminders

- Hash passwords with bcrypt (cost factor 12+). Never store plaintext.
- Use `secrets.token_urlsafe(32)` for generating random keys and tokens.
- Set short access token TTLs (15-30 min). Use refresh tokens (7-30 days) for re-authentication.
- Store refresh tokens in the database so they can be revoked.
- Always validate token `type` claim to prevent access tokens being used as refresh tokens and vice versa.
- Rate-limit login endpoints to prevent brute force.
