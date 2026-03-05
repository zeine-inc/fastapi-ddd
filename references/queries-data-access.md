# Queries & Data Access

## Why NOT Repository Pattern

SQLAlchemy's `AsyncSession` already implements Unit of Work + Repository:

- `session.add()` = repository.save()
- `session.execute(select(...))` = repository.find()
- `session.commit()` / `session.rollback()` = unit of work

Adding a Repository class on top creates a **double abstraction**:

```python
# ❌ Over-engineered — wraps what SQLAlchemy already does
class UserRepository:
    def __init__(self, db: AsyncSession):
        self.db = db

    async def get_by_id(self, user_id: UUID) -> User | None:
        result = await self.db.execute(select(User).where(User.id == user_id))
        return result.scalar_one_or_none()

    async def create(self, user: User) -> User:
        self.db.add(user)
        await self.db.flush()
        return user

# This adds a file per module, doubles the code, and provides no benefit.
# You'll never swap SQLAlchemy for another ORM.
```

### FAQ: "But What About Testability?"

The most common argument for Repository is testability: "I can mock the repository in tests." Here's why that argument doesn't hold in FastAPI:

**1. Use a real test database, not mocks.**

Mocking `AsyncSession` is fragile — you're reimplementing SQLAlchemy's behavior with `MagicMock`. Real queries catch SQL bugs, constraint violations, and N+1 problems. Mocks don't.

```python
# ✅ Test with real database — catches real bugs
@pytest.mark.anyio
async def test_upload_creates_document(db: AsyncSession):
    service = DocumentService(db)
    result = await service.upload(file, user_id=uuid4())
    assert result.filename == "test.pdf"

# ❌ Mock-based test — tests nothing useful
async def test_upload(mock_db):
    mock_db.execute.return_value = MockResult(...)  # Fragile, doesn't test SQL
```

The `conftest.py` fixture with SQLite in-memory or a test PostgreSQL instance is the standard approach. See `references/testing.md` for the full setup.

**2. `queries.py` enables surgical mocking when needed.**

If a service calls `count_docs_this_month()` and you want to test the service logic without hitting the database for that specific query, you can mock just that function:

```python
from unittest.mock import patch

@pytest.mark.anyio
async def test_service_checks_limit(db):
    with patch("app.modules.subscriptions.queries.count_docs_this_month", return_value=50):
        # Test the service's reaction to the limit being reached
        ...
```

This is more granular than mocking an entire Repository class and doesn't require an interface/protocol.

**3. When IS Repository acceptable?**

Repository is justified in exactly three scenarios — and none of them apply to typical FastAPI projects:

| Scenario | Why Repository helps | How common |
| --- | --- | --- |
| **Multi-database routing** (read replica + primary) | Encapsulates which session to use per operation | Rare — use when you genuinely have 2+ databases |
| **CQRS with separate read/write models** | Read repository uses denormalized views, write repository uses normalized tables | Very rare in FastAPI |
| **Published library/SDK** | Consumers provide their own persistence implementation | Only for library authors |

If none of these apply, skip Repository. SQLAlchemy + `queries.py` gives you everything you need.

## The Right Abstraction: `queries.py`

For **complex or reused queries**, extract them into named functions in `queries.py`:

```python
# app/modules/subscriptions/queries.py
from sqlalchemy import select, func
from sqlalchemy.ext.asyncio import AsyncSession
from uuid import UUID

from app.modules.documents.models import Document


async def count_docs_this_month(
    db: AsyncSession,
    user_id: UUID,
    billing_cycle_start: int,
) -> int:
    """Count non-deleted documents uploaded since billing cycle start."""
    result = await db.execute(
        select(func.count(Document.id)).where(
            Document.user_id == user_id,
            Document.is_deleted == False,
            Document.created_at >= billing_cycle_start,
        )
    )
    return result.scalar_one()


async def sum_storage_bytes(db: AsyncSession, user_id: UUID) -> int:
    """Total storage used by non-deleted documents."""
    result = await db.execute(
        select(func.coalesce(func.sum(Document.file_size), 0)).where(
            Document.user_id == user_id,
            Document.is_deleted == False,
        )
    )
    return result.scalar_one()


async def count_active_sessions(db: AsyncSession, user_id: UUID) -> int:
    """Count non-deleted chat sessions for the user."""
    result = await db.execute(
        select(func.count()).select_from(AgnoSession).where(
            AgnoSession.user_id == str(user_id),
            AgnoSession.is_deleted == False,
        )
    )
    return result.scalar_one()
```

## When to Use `queries.py` vs Inline

| Situation                                             | Approach          |
| ----------------------------------------------------- | ----------------- |
| Simple CRUD (get by id, create, update)               | Inline in service |
| Query used by multiple callers (service + worker)     | `queries.py`      |
| Complex query (joins, aggregations, subqueries)       | `queries.py`      |
| Query with business semantics (count_docs_this_month) | `queries.py`      |
| One-off filter query                                  | Inline in service |

```python
# ✅ Inline — simple, used once
async def get_by_id(self, doc_id: UUID, user_id: UUID) -> Document | None:
    result = await self.db.execute(
        select(Document).where(
            Document.id == doc_id,
            Document.user_id == user_id,
            Document.is_deleted == False,
        )
    )
    return result.scalar_one_or_none()

# ✅ In queries.py — complex, reused, has business semantics
# See count_docs_this_month above
```

## Sharing Queries Between Async and Sync

When a background worker (Celery) needs the same query as the async service:

### Option A: Query Builder Functions

Return the SQLAlchemy statement, let the caller execute it:

```python
# app/modules/documents/queries.py
from sqlalchemy import select, func
from app.modules.documents.models import Document


def build_file_count_query(user_id: UUID):
    """Build the file count query — can be executed by async or sync session."""
    return (
        select(
            func.count(Document.id).label("total"),
            func.count(Document.id).filter(Document.ocr_status == "pending").label("pending"),
            func.count(Document.id).filter(Document.ocr_status == "completed").label("completed"),
        )
        .where(Document.user_id == user_id, Document.is_deleted == False)
    )


# Async service:
result = await db.execute(build_file_count_query(user_id))

# Sync worker:
result = sync_session.execute(build_file_count_query(user_id))
```

### Option B: Dual Functions

When async/sync need slightly different handling:

```python
# Async version (in service)
async def get_file_counts(db: AsyncSession, user_id: UUID) -> dict:
    result = await db.execute(build_file_count_query(user_id))
    row = result.one()
    return {"total": row.total, "pending": row.pending, "completed": row.completed}

# Sync version (in worker)
def get_file_counts_sync(session: Session, user_id: UUID) -> dict:
    result = session.execute(build_file_count_query(user_id))
    row = result.one()
    return {"total": row.total, "pending": row.pending, "completed": row.completed}
```

## Multi-Tenancy Pattern

Every query that touches user-scoped data **must** include `user_id`:

```python
# ✅ SAFE — always scoped
select(Document).where(
    Document.id == doc_id,
    Document.user_id == user_id,       # Multi-tenant isolation
    Document.is_deleted == False,       # Soft delete
)

# ❌ DANGEROUS — no user scoping
select(Document).where(Document.id == doc_id)
# Any user could access any document by guessing UUIDs
```

**Where `user_id` comes from**: Always from `get_current_user` (JWT-validated), never from request parameters:

```python
# ✅ user_id from JWT
@router.get("/{doc_id}")
async def get_doc(doc_id: UUID, current_user: CurrentUser, db: DB):
    return await service.get_by_id(doc_id, user_id=current_user.id)

# ❌ user_id from request — client can forge this
@router.get("/{doc_id}")
async def get_doc(doc_id: UUID, user_id: UUID, db: DB):
    return await service.get_by_id(doc_id, user_id=user_id)
```

## Soft Delete Pattern

Use a boolean `is_deleted` column instead of physical deletion:

```python
# Model
class Document(Base):
    is_deleted: Mapped[bool] = mapped_column(default=False, index=True)

# Service — soft delete
async def soft_delete(self, doc_id: UUID, user_id: UUID) -> None:
    doc = await self._get_owned_document(doc_id, user_id)
    doc.is_deleted = True
    await self.db.flush()

# ALL queries must exclude deleted records
select(Document).where(
    Document.user_id == user_id,
    Document.is_deleted == False,  # Always include this
)
```

If you find yourself writing `is_deleted == False` everywhere, consider a helper:

```python
def active_documents(user_id: UUID):
    """Base filter for non-deleted documents owned by user."""
    return select(Document).where(
        Document.user_id == user_id,
        Document.is_deleted == False,
    )

# Usage
query = active_documents(user_id).where(Document.ocr_status == "completed")
```

## Transaction Management

Let the `get_db` dependency handle transactions — don't commit in services:

```python
# app/core/database.py
async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session() as session:
        try:
            yield session
            await session.commit()       # Auto-commit on success
        except Exception:
            await session.rollback()     # Auto-rollback on error
            raise

# Service — just use flush(), never commit()
async def upload(self, file: UploadFile, user_id: UUID):
    document = Document(...)
    self.db.add(document)
    await self.db.flush()       # Write to DB but don't commit yet
    await self.db.refresh(document)  # Get generated values (id, timestamps)
    return document
    # commit() happens automatically in get_db when the request succeeds
```

Use `flush()` in services to get generated values (like auto-increment IDs) without committing the transaction. The transaction commits when the request completes successfully.

### Savepoints for Long Operations

When a request involves multiple independent operations (e.g., upload file + create record + trigger indexing), and a late failure should not roll back early work, use savepoints:

```python
async def upload_and_process(self, file: UploadFile, user_id: UUID):
    # Step 1: Create record (want to keep even if step 2 fails)
    document = Document(user_id=user_id, filename=file.filename, ocr_status="pending")
    self.db.add(document)
    await self.db.flush()

    # Step 2: External call that might fail — use savepoint
    async with self.db.begin_nested():  # Creates a SAVEPOINT
        try:
            await self.storage.upload_file(file, str(document.id))
            document.file_path = f"uploads/{document.id}"
            await self.db.flush()
        except StorageError:
            # Savepoint rolls back, but document record survives
            document.ocr_status = "upload_failed"
            await self.db.flush()
```

**When to use savepoints**: Only when a request has multiple independent steps where partial success is acceptable. For typical CRUD, the per-request transaction is sufficient.

## Eager Loading for Nested Responses

When response schemas include nested objects (e.g., `DocumentResponse` with `author: AuthorResponse`), you **must** use eager loading to avoid N+1 queries:

```python
# ❌ N+1 — each doc.author triggers a separate query
result = await db.execute(select(Document).where(Document.user_id == user_id))
documents = result.scalars().all()
# Iterating and accessing doc.author = N additional queries

# ✅ Eager loaded — single query with JOIN
from sqlalchemy.orm import selectinload, joinedload

# Use joinedload for many-to-one (each document has ONE author)
result = await db.execute(
    select(Document)
    .where(Document.user_id == user_id)
    .options(joinedload(Document.author))
)
documents = result.scalars().unique().all()

# Use selectinload for one-to-many (each author has MANY posts)
result = await db.execute(
    select(Author)
    .where(Author.id == author_id)
    .options(selectinload(Author.posts))
)
```

### Which Loading Strategy to Use

| Relationship                     | Strategy                                                         | Why                                                      |
| -------------------------------- | ---------------------------------------------------------------- | -------------------------------------------------------- |
| Many-to-one (`doc.author`)       | `joinedload()`                                                   | Single query with LEFT JOIN, efficient for single parent |
| One-to-many (`author.posts`)     | `selectinload()`                                                 | Separate IN query, avoids cartesian product explosion    |
| Many-to-many                     | `selectinload()`                                                 | Same reason — avoids row multiplication                  |
| Deeply nested (`doc.author.org`) | Chain: `.options(joinedload(Doc.author).joinedload(Author.org))` | Load full chain in minimal queries                       |

### In `queries.py`

Extract eager-loaded queries when reused:

```python
# app/modules/documents/queries.py
from sqlalchemy.orm import joinedload

def documents_with_author(user_id: UUID):
    """Documents with author pre-loaded — avoids N+1 for list endpoints."""
    return (
        select(Document)
        .where(Document.user_id == user_id, Document.is_deleted == False)
        .options(joinedload(Document.author))
        .order_by(Document.created_at.desc())
    )
```

**Rule**: If your response schema has nested objects, your query **must** have matching `.options()`. No exceptions.

## Bulk Operations

For importing data, batch processing, or any operation touching many rows, use SQLAlchemy's bulk APIs instead of looping with `session.add()`:

### Bulk Insert

```python
from sqlalchemy import insert

# ❌ SLOW — N round-trips to the database
for item in items:
    db.add(Document(user_id=user_id, filename=item["filename"]))
await db.flush()

# ✅ FAST — single INSERT with multiple rows
await db.execute(
    insert(Document),
    [
        {"user_id": user_id, "filename": item["filename"], "ocr_status": "pending"}
        for item in items
    ],
)
await db.flush()
```

### Bulk Update

```python
from sqlalchemy import update

# ❌ SLOW — loads every object, updates one by one
docs = (await db.execute(select(Document).where(...))).scalars().all()
for doc in docs:
    doc.ocr_status = "canceled"
await db.flush()

# ✅ FAST — single UPDATE statement
await db.execute(
    update(Document)
    .where(Document.user_id == user_id, Document.ocr_status == "pending")
    .values(ocr_status="canceled")
)
await db.flush()
```

### Bulk Delete (Soft)

```python
# ✅ Soft-delete multiple records in a single statement
await db.execute(
    update(Document)
    .where(Document.user_id == user_id, Document.id.in_(document_ids))
    .values(is_deleted=True)
)
await db.flush()
```

### Upsert (Insert or Update)

PostgreSQL-specific — useful for sync/import operations:

```python
from sqlalchemy.dialects.postgresql import insert as pg_insert

stmt = pg_insert(Document).values(
    [{"id": d["id"], "user_id": user_id, "filename": d["filename"]} for d in data]
)
stmt = stmt.on_conflict_do_update(
    index_elements=["id"],
    set_={"filename": stmt.excluded.filename, "updated_at": func.now()},
)
await db.execute(stmt)
await db.flush()
```

### When to Use Bulk Operations

| Scenario                         | Approach                                   |
| -------------------------------- | ------------------------------------------ |
| Creating 1-5 records             | `db.add()` in a loop is fine               |
| Creating 10+ records             | `insert().values([...])`                   |
| Updating a filtered set          | `update().where().values()`                |
| Import/sync from external source | Bulk insert + upsert                       |
| Mass soft-delete                 | `update().where().values(is_deleted=True)` |

**Caveat**: Bulk operations bypass ORM events (`after_insert`, `after_update`). If you rely on ORM events for side effects, use the ORM loop instead.

## Connection Pool Configuration

The async engine connection pool directly impacts concurrency and reliability. Configure it based on your deployment:

```python
# app/core/database.py
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession

engine = create_async_engine(
    settings.database_url,

    # Pool sizing — match your expected concurrency
    pool_size=20,           # Persistent connections (default: 5)
    max_overflow=10,        # Extra connections under load (default: 10)
    # Total max connections = pool_size + max_overflow = 30

    # Timeouts
    pool_timeout=30,        # Seconds to wait for a connection from pool (default: 30)
    pool_recycle=1800,      # Recycle connections after 30 min (prevents stale connections)
    pool_pre_ping=True,     # Verify connection is alive before using (catches disconnects)

    # Performance
    echo=settings.debug,    # Log SQL in development only
)

async_session = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)
```

### Pool Sizing Guidelines

| Deployment                | `pool_size` | `max_overflow` | Why                                          |
| ------------------------- | ----------- | -------------- | -------------------------------------------- |
| Development (1 worker)    | 5           | 5              | Low concurrency                              |
| Production (2-4 workers)  | 10-20       | 10             | Match concurrent requests per worker         |
| High traffic (8+ workers) | 5-10        | 5-10           | Per-worker pool, total = workers x pool_size |

**Critical constraint**: `workers * (pool_size + max_overflow)` must not exceed your database's `max_connections`. PostgreSQL default is 100.

```
Example: 4 uvicorn workers × (20 pool_size + 10 overflow) = 120 connections
→ EXCEEDS PostgreSQL default of 100! Reduce pool_size or use PgBouncer.
```

### `pool_pre_ping` — Always Enable in Production

Without `pool_pre_ping`, a connection that was closed by the database (idle timeout, failover) will cause a `ConnectionError` on first use. `pool_pre_ping=True` sends a lightweight `SELECT 1` before handing the connection to your code.

### `expire_on_commit=False` — Required for Async

Without this, accessing any ORM attribute after `commit()` triggers a lazy load — which fails in async because lazy loads are synchronous. Always set `expire_on_commit=False` in `async_sessionmaker`.

### Production Checklist

These two settings are **non-negotiable** — omitting either one causes silent failures in production:

| Setting                  | What Happens Without It                                                           | Risk                            |
| ------------------------ | --------------------------------------------------------------------------------- | ------------------------------- |
| `pool_pre_ping=True`     | Stale connections cause `ConnectionError` on first use after DB restart/failover  | 500 errors until pool refreshes |
| `expire_on_commit=False` | ORM attribute access after commit triggers synchronous lazy load in async context | `MissingGreenlet` crash         |

Always verify both are set in `core/database.py`. See `references/performance.md` for additional async configuration guidance.
