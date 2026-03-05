# Testing Patterns

## Test Structure

Mirror the module structure in your test directory:

```
tests/
├── conftest.py              # Shared fixtures: db session, test client, factories
├── modules/
│   ├── documents/
│   │   ├── test_service.py  # Business logic tests
│   │   ├── test_router.py   # HTTP integration tests
│   │   ├── test_guards.py   # Guard/dependency tests
│   │   └── test_queries.py  # Query function tests
│   ├── billing/
│   │   └── ...
│   └── auth/
│       └── ...
└── core/
    └── test_security.py     # Auth utility tests
```

## Fixtures — Database Session

Use an async test session with transaction rollback for isolation:

```python
# tests/conftest.py
import pytest
from httpx import AsyncClient, ASGITransport
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession

from app.core.database import Base, get_db
from app.main import create_app

TEST_DATABASE_URL = "sqlite+aiosqlite:///./test.db"
# Or for PostgreSQL: "postgresql+asyncpg://user:pass@localhost:5432/test_db"

engine = create_async_engine(TEST_DATABASE_URL)
TestSession = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)


@pytest.fixture(autouse=True)
async def setup_db():
    """Create tables before each test, drop after."""
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)


@pytest.fixture
async def db():
    """Provide a test database session with auto-rollback."""
    async with TestSession() as session:
        yield session


@pytest.fixture
async def client(db: AsyncSession):
    """Provide an async HTTP test client with DI overrides."""
    app = create_app()
    app.dependency_overrides[get_db] = lambda: db

    async with AsyncClient(
        transport=ASGITransport(app=app),
        base_url="http://test",
    ) as client:
        yield client
```

## Testing Services

Services are the core of your business logic — test them directly with a real database session:

```python
# tests/modules/documents/test_service.py
import pytest
from uuid import uuid4
from unittest.mock import AsyncMock, patch

from app.modules.documents.service import DocumentService
from app.modules.documents.models import Document
from app.shared.exceptions import NotFoundError


@pytest.mark.anyio
async def test_upload_creates_document(db):
    service = DocumentService(db)
    file = make_upload_file("test.pdf", content_type="application/pdf", size=1024)

    result = await service.upload(file, user_id=uuid4())

    assert result.filename == "test.pdf"
    assert result.ocr_status == "pending"


@pytest.mark.anyio
async def test_get_by_id_returns_document(db):
    user_id = uuid4()
    doc = Document(user_id=user_id, filename="test.pdf", ocr_status="completed")
    db.add(doc)
    await db.flush()

    service = DocumentService(db)
    result = await service.get_by_id(doc.id, user_id=user_id)

    assert result.id == doc.id
    assert result.filename == "test.pdf"


@pytest.mark.anyio
async def test_get_by_id_raises_not_found(db):
    service = DocumentService(db)

    with pytest.raises(NotFoundError):
        await service.get_by_id(uuid4(), user_id=uuid4())


@pytest.mark.anyio
async def test_soft_delete_marks_as_deleted(db):
    user_id = uuid4()
    doc = Document(user_id=user_id, filename="test.pdf")
    db.add(doc)
    await db.flush()

    service = DocumentService(db)
    await service.soft_delete(doc.id, user_id=user_id)

    await db.refresh(doc)
    assert doc.is_deleted is True


@pytest.mark.anyio
async def test_list_excludes_other_users_documents(db):
    user_a, user_b = uuid4(), uuid4()
    db.add(Document(user_id=user_a, filename="a.pdf"))
    db.add(Document(user_id=user_b, filename="b.pdf"))
    await db.flush()

    service = DocumentService(db)
    result = await service.list_files(user_id=user_a)

    assert len(result.items) == 1
    assert result.items[0].filename == "a.pdf"
```

**Key patterns**:

- Test with real `AsyncSession` (not mocks) — catches SQL errors, constraint violations
- Each test creates its own data — no shared state between tests
- Test multi-tenancy isolation (user A can't see user B's data)
- Test domain exceptions (not HTTP status codes)

## Testing Guards

Guards are FastAPI dependencies — test them as regular async functions:

```python
# tests/modules/billing/test_guards.py
import pytest
from unittest.mock import AsyncMock
from uuid import uuid4

from app.modules.subscriptions.guards import require_active_plan, check_doc_limit
from app.modules.users.models import User
from fastapi import HTTPException


def make_user(**overrides) -> User:
    """Factory for test users."""
    defaults = {
        "id": uuid4(),
        "email": "test@example.com",
        "plan": "starter",
        "plan_status": "active",
    }
    defaults.update(overrides)
    user = User.__new__(User)
    for k, v in defaults.items():
        setattr(user, k, v)
    return user


@pytest.mark.anyio
async def test_require_active_plan_passes_for_active_user(db):
    user = make_user(plan_status="active")
    result = await require_active_plan(current_user=user, db=db)
    assert result == user


@pytest.mark.anyio
async def test_require_active_plan_rejects_expired_user(db):
    user = make_user(plan_status="expired")
    with pytest.raises(HTTPException) as exc_info:
        await require_active_plan(current_user=user, db=db)
    assert exc_info.value.status_code == 402


@pytest.mark.anyio
async def test_check_doc_limit_passes_under_limit(db):
    user = make_user(plan="starter")
    # Pre-populate with fewer docs than the limit
    # ...
    result = await check_doc_limit(current_user=user, db=db)
    assert result == user
```

**Key patterns**:

- Use a `make_user()` factory — create User objects without database overhead
- Test both pass and fail paths for each guard
- Test the HTTP status code and error code in the detail

## Testing Queries

Test query functions against a real database to catch SQL errors:

```python
# tests/modules/documents/test_queries.py
import pytest
from uuid import uuid4
from datetime import datetime, timedelta

from app.modules.subscriptions.queries import count_docs_this_month
from app.modules.documents.models import Document


@pytest.mark.anyio
async def test_count_docs_this_month_counts_correctly(db):
    user_id = uuid4()
    cycle_start = datetime.utcnow() - timedelta(days=15)

    # 2 docs this month
    db.add(Document(user_id=user_id, filename="a.pdf", created_at=datetime.utcnow()))
    db.add(Document(user_id=user_id, filename="b.pdf", created_at=datetime.utcnow()))
    # 1 doc before cycle
    db.add(Document(user_id=user_id, filename="old.pdf", created_at=cycle_start - timedelta(days=5)))
    # 1 deleted doc (should not count)
    db.add(Document(user_id=user_id, filename="del.pdf", is_deleted=True, created_at=datetime.utcnow()))
    await db.flush()

    count = await count_docs_this_month(db, user_id, billing_cycle_start=int(cycle_start.timestamp()))
    assert count == 2


@pytest.mark.anyio
async def test_count_docs_isolates_by_user(db):
    user_a, user_b = uuid4(), uuid4()
    db.add(Document(user_id=user_a, filename="a.pdf"))
    db.add(Document(user_id=user_b, filename="b.pdf"))
    await db.flush()

    cycle_start = int((datetime.utcnow() - timedelta(days=30)).timestamp())
    assert await count_docs_this_month(db, user_a, billing_cycle_start=cycle_start) == 1
    assert await count_docs_this_month(db, user_b, billing_cycle_start=cycle_start) == 1
```

## Testing Query Builders (Statement Inspection)

For `build_*_query()` functions that return SQLAlchemy statements, you can inspect the SQL without executing:

```python
from app.modules.documents.queries import build_file_count_query

def test_build_file_count_query_includes_user_filter():
    stmt = build_file_count_query(uuid4())
    sql = str(stmt.compile(compile_kwargs={"literal_binds": True}))
    assert "user_id" in sql
    assert "is_deleted" in sql
```

## Testing Routers (HTTP Integration)

Test the full HTTP flow — request parsing, DI, service, response:

```python
# tests/modules/documents/test_router.py
import pytest
from uuid import uuid4
from app.core.security import get_current_user
from app.modules.users.models import User


@pytest.fixture
def auth_user():
    """Override get_current_user to return a test user."""
    user = make_user(plan_status="active")
    return user


@pytest.fixture
async def authed_client(client, auth_user):
    """Client with auth dependency overridden."""
    from app.main import create_app
    app = create_app()
    app.dependency_overrides[get_current_user] = lambda: auth_user
    # ... (rebuild client with overrides)
    return client


@pytest.mark.anyio
async def test_list_documents_returns_200(authed_client):
    response = await authed_client.get("/documents/")
    assert response.status_code == 200
    data = response.json()
    assert "items" in data
    assert "total" in data


@pytest.mark.anyio
async def test_get_nonexistent_document_returns_404(authed_client):
    response = await authed_client.get(f"/documents/{uuid4()}")
    assert response.status_code == 404
```

## What to Test at Each Layer

| Layer   | Test Type        | What to Assert                                                |
| ------- | ---------------- | ------------------------------------------------------------- |
| Service | Unit/Integration | Business logic, data transformations, domain exceptions       |
| Guards  | Unit             | Pass/fail conditions, correct HTTP status codes               |
| Queries | Integration      | Correct SQL results, filter correctness, multi-tenancy        |
| Router  | Integration      | HTTP status codes, response shapes, auth enforcement          |
| Schemas | Unit             | Validation rules, `exclude_unset` behavior, `from_attributes` |

## Testing Anti-Patterns

```python
# ❌ Mocking the database session — hides real SQL bugs
async def test_upload(mock_db):
    mock_db.execute.return_value = MockResult(...)  # Fragile, doesn't test SQL

# ✅ Use a real test database — catches constraint violations, bad queries
async def test_upload(db):
    service = DocumentService(db)
    result = await service.upload(...)

# ❌ Testing implementation details
assert mock_db.add.called_with(Document(...))  # Fragile, breaks on refactor

# ✅ Testing outcomes
doc = await db.get(Document, result.id)
assert doc.filename == "test.pdf"

# ❌ Sharing state between tests
test_user = User(...)  # Module-level — shared across tests

# ✅ Create fresh data per test
async def test_something(db):
    user = make_user()  # Factory creates fresh instance
```

## Configuration

Use `pytest` with `anyio` for async tests:

```toml
# pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"

[tool.pytest]
markers = ["anyio: mark test as async"]
```

Required packages: `pytest`, `pytest-anyio`, `httpx` (for `AsyncClient`), `aiosqlite` (for SQLite async in tests).
