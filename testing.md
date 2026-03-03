# Testing: Async Fixtures, Mocking Dependencies, Integration Tests

## Test Setup

Install test dependencies:

```
pytest
pytest-asyncio
httpx
factory-boy          # Optional: for model factories
```

### conftest.py

```python
# tests/conftest.py
import pytest
from httpx import ASGITransport, AsyncClient
from sqlalchemy.ext.asyncio import (
    AsyncSession,
    async_sessionmaker,
    create_async_engine,
)
from app.database import Base
from app.main import create_app
from app.dependencies import get_db
from app.config import settings

TEST_DATABASE_URL = settings.DATABASE_URL.replace("/mydb", "/mydb_test")

engine_test = create_async_engine(TEST_DATABASE_URL, echo=False)
async_session_test = async_sessionmaker(
    engine_test, class_=AsyncSession, expire_on_commit=False,
)

@pytest.fixture(scope="session")
def anyio_backend():
    return "asyncio"

@pytest.fixture(scope="session", autouse=True)
async def setup_database():
    """Create all tables once for the test session."""
    async with engine_test.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield
    async with engine_test.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
    await engine_test.dispose()

@pytest.fixture
async def db_session():
    """Per-test session with transaction rollback."""
    async with engine_test.connect() as conn:
        transaction = await conn.begin()
        session = AsyncSession(bind=conn, expire_on_commit=False)
        try:
            yield session
        finally:
            await transaction.rollback()
            await session.close()

@pytest.fixture
async def client(db_session: AsyncSession):
    """HTTP client with DB dependency overridden."""
    app = create_app()

    async def override_get_db():
        yield db_session

    app.dependency_overrides[get_db] = override_get_db

    async with AsyncClient(
        transport=ASGITransport(app=app),
        base_url="http://test",
    ) as ac:
        yield ac

    app.dependency_overrides.clear()
```

### pytest configuration

```toml
# pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
filterwarnings = ["ignore::DeprecationWarning"]
```

## Writing Integration Tests

Test the full request cycle through the API:

```python
# tests/test_users/test_create_user.py
import pytest
from httpx import AsyncClient

async def test_create_user_success(client: AsyncClient, auth_headers: dict):
    response = await client.post(
        "/api/v1/users/",
        json={"email": "new@example.com", "name": "New User"},
        headers=auth_headers,
    )
    assert response.status_code == 201
    data = response.json()
    assert data["email"] == "new@example.com"
    assert data["name"] == "New User"
    assert "id" in data

async def test_create_user_duplicate_email(client: AsyncClient, auth_headers: dict):
    payload = {"email": "dup@example.com", "name": "User"}
    await client.post("/api/v1/users/", json=payload, headers=auth_headers)

    response = await client.post(
        "/api/v1/users/",
        json=payload,
        headers=auth_headers,
    )
    assert response.status_code == 409
    assert response.json()["error"]["code"] == "DUPLICATE_ENTITY"

async def test_create_user_validation_error(client: AsyncClient, auth_headers: dict):
    response = await client.post(
        "/api/v1/users/",
        json={"email": "not-an-email", "name": ""},
        headers=auth_headers,
    )
    assert response.status_code == 422
    assert response.json()["error"]["code"] == "VALIDATION_ERROR"
```

## Auth Test Fixtures

```python
# tests/conftest.py (continued)
from app.auth.service import create_access_token, hash_password
from app.users.models import User

@pytest.fixture
async def test_user(db_session: AsyncSession) -> User:
    user = User(
        email="test@example.com",
        name="Test User",
        hashed_password=hash_password("testpass123"),
        role="admin",
    )
    db_session.add(user)
    await db_session.flush()
    return user

@pytest.fixture
def auth_headers(test_user: User) -> dict:
    token = create_access_token(str(test_user.id))
    return {"Authorization": f"Bearer {token}"}
```

## Unit Testing Services

Test service logic directly without HTTP overhead:

```python
# tests/test_users/test_user_service.py
import pytest
from app.users.service import UserService
from app.users.schemas import UserCreate
from app.exceptions import NotFoundError, DuplicateEntityError

async def test_get_user_not_found(db_session):
    service = UserService(db_session)
    with pytest.raises(NotFoundError):
        await service.get_user(99999)

async def test_create_user_service(db_session):
    service = UserService(db_session)
    data = UserCreate(email="svc@example.com", name="Svc User")
    user = await service.create_user(data)
    assert user.id is not None
    assert user.email == "svc@example.com"
```

## Mocking External Services

Override dependencies, not internal functions. This keeps tests realistic.

```python
# Mock an external payment service
from app.payments.dependencies import get_payment_client

class MockPaymentClient:
    async def charge(self, amount: int, token: str):
        return {"id": "ch_mock_123", "status": "succeeded"}

@pytest.fixture
async def client_with_mock_payments(db_session):
    app = create_app()
    app.dependency_overrides[get_db] = lambda: db_session
    app.dependency_overrides[get_payment_client] = lambda: MockPaymentClient()

    async with AsyncClient(
        transport=ASGITransport(app=app), base_url="http://test"
    ) as ac:
        yield ac

    app.dependency_overrides.clear()
```

For HTTP mocks (third-party APIs), use `respx` with `httpx`:

```python
import respx

@respx.mock
async def test_external_api_call(client):
    respx.get("https://api.external.com/data").respond(
        json={"result": "ok"}, status_code=200,
    )
    response = await client.get("/api/v1/proxy-endpoint")
    assert response.status_code == 200
```

## Test Organization

```
tests/
├── conftest.py              # Shared fixtures (db, client, auth)
├── factories.py             # Model factories (optional, with factory-boy)
├── test_auth/
│   ├── test_login.py
│   └── test_permissions.py
├── test_users/
│   ├── test_create_user.py
│   ├── test_get_user.py
│   └── test_user_service.py  # Unit tests for service layer
└── test_health.py
```

**Naming:** `test_<action>_<scenario>.py` for files, `test_<behavior>` for functions. Be descriptive — `test_create_user_with_duplicate_email_returns_409` is better than `test_create_user_2`.

## Running Tests

```bash
# Run all tests
pytest

# Run with coverage
pytest --cov=app --cov-report=term-missing

# Run a specific test file
pytest tests/test_users/test_create_user.py -v

# Run tests matching a pattern
pytest -k "duplicate_email" -v
```
