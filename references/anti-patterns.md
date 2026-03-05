# Anti-Patterns — The Over-Engineering & Under-Engineering Scale

Good architecture sits between two failure modes. This reference helps identify both.

## Over-Engineering (Too Much Architecture)

### Repository Layer Over SQLAlchemy

```python
# ❌ OVER-ENGINEERED — Repository wrapping what SQLAlchemy already does
class UserRepository:
    def __init__(self, session: AsyncSession):
        self.session = session

    async def get_by_id(self, id: UUID) -> User | None:
        result = await self.session.execute(select(User).where(User.id == id))
        return result.scalar_one_or_none()

    async def save(self, user: User) -> User:
        self.session.add(user)
        await self.session.flush()
        return user

# Then the service wraps the repository:
class UserService:
    def __init__(self, repo: UserRepository):
        self.repo = repo

    async def get_user(self, id: UUID) -> User:
        return await self.repo.get_by_id(id)  # Just proxying!
```

**Why it's wrong**: SQLAlchemy's `AsyncSession` IS a repository + unit of work. Adding a layer on top doubles the code per module (service + repository + tests for both). You will never swap SQLAlchemy for another ORM.

**What to do instead**: Use the service directly with the session. Extract complex queries to `queries.py`.

### Domain Events Bus for Small Apps

```python
# ❌ OVER-ENGINEERED — Full event bus for 3 modules
class EventBus:
    _handlers: dict[str, list[Callable]] = {}

    @classmethod
    def subscribe(cls, event_type: str, handler: Callable):
        cls._handlers.setdefault(event_type, []).append(handler)

    @classmethod
    async def publish(cls, event_type: str, data: dict):
        for handler in cls._handlers.get(event_type, []):
            await handler(data)

# Registration at startup
EventBus.subscribe("user.created", handle_user_created_billing)
EventBus.subscribe("user.created", handle_user_created_email)
EventBus.subscribe("user.created", handle_user_created_analytics)
```

**Why it's wrong**: For < 5 modules with simple event flows, an event bus adds indirection (harder to trace "what happens when a user is created?"), implicit coupling (handlers registered at startup, invisible to the reader), and a testing burden (mocking the bus, verifying subscriptions).

**What to do instead**: Router orchestration — the router explicitly calls services in sequence. When you need fire-and-forget, use `BackgroundTasks`.

### CQRS for Simple CRUD

```python
# ❌ OVER-ENGINEERED — Separate read/write models for a basic CRUD app
class CreateUserCommand:
    email: str
    name: str

class UserCreatedEvent:
    user_id: UUID

class UserReadModel:
    id: UUID
    email: str
    name: str

class UserCommandHandler:
    async def handle(self, cmd: CreateUserCommand) -> UserCreatedEvent: ...

class UserQueryHandler:
    async def handle(self, user_id: UUID) -> UserReadModel: ...
```

**Why it's wrong**: CQRS is for systems where reads and writes have fundamentally different shapes, scales, or consistency requirements. Most FastAPI apps read and write the same tables with the same models.

**What to do instead**: Standard service with methods for both reads and writes. Same ORM model, different Pydantic schemas for input/output.

### Abstract Base Classes for Everything

```python
# ❌ OVER-ENGINEERED — ABCs that will only ever have one implementation
from abc import ABC, abstractmethod

class IUserService(ABC):
    @abstractmethod
    async def get_user(self, id: UUID) -> User: ...

    @abstractmethod
    async def create_user(self, data: dict) -> User: ...

class UserService(IUserService):
    async def get_user(self, id: UUID) -> User:
        ...  # The only implementation that will ever exist
```

**Why it's wrong**: Interfaces are for when you have multiple implementations. In Python, duck typing makes ABCs rarely necessary. One implementation = no interface needed.

---

## Under-Engineering (Too Little Architecture)

### Raw SQL in Route Handlers

```python
# ❌ UNDER-ENGINEERED — SQL, business logic, and HTTP all in one function
@router.get("/sessions")
async def list_sessions(request: Request):
    user_id = request.state.user_id
    engine = _get_engine()
    with engine.connect() as conn:
        rows = conn.execute(text("""
            SELECT session_id, agent_id, created_at
            FROM agno_sessions
            WHERE user_id = :uid AND is_deleted = false
            ORDER BY created_at DESC
        """), {"uid": user_id})
        sessions = [dict(row._mapping) for row in rows]
    return {"sessions": sessions}
```

**Why it's wrong**: Untestable (needs real DB), no type safety, no schema validation, SQL injection risk with string concatenation, impossible to reuse the query.

**What to do instead**: Extract to service + queries. Use ORM or parameterized queries. Return Pydantic schemas.

### Business Logic in Routers

```python
# ❌ UNDER-ENGINEERED — Router makes business decisions
@router.post("/upload")
async def upload(file: UploadFile, db: DB, current_user: CurrentUser):
    # Validation logic in router
    if file.size > 10_000_000:
        raise HTTPException(400, "File too large")
    if file.content_type not in ["application/pdf", "image/png"]:
        raise HTTPException(400, "Invalid file type")

    # Storage logic in router
    path = f"uploads/{current_user.id}/{file.filename}"
    await minio_client.put_object(bucket, path, file.file, file.size)

    # Database logic in router
    doc = Document(user_id=current_user.id, filename=file.filename, file_path=path)
    db.add(doc)
    await db.flush()

    # Background task dispatch in router
    process_ocr.delay(str(doc.id))

    return {"id": str(doc.id), "filename": file.filename}
```

**Why it's wrong**: The router does everything — validation, storage, persistence, task dispatch. Can't test business logic without HTTP. Can't reuse upload logic (e.g., from a CLI tool or worker).

**What to do instead**: Router calls `DocumentService.upload()`. Service handles all the steps. Router only parses the request and returns the response.

### Returning Raw Dicts

```python
# ❌ UNDER-ENGINEERED — No type safety, no validation, no docs
@router.get("/documents/{doc_id}")
async def get_document(doc_id: UUID, db: DB):
    doc = await db.get(Document, doc_id)
    return {
        "id": str(doc.id),
        "filename": doc.filename,
        "status": doc.ocr_status,
        "result": doc.ocr_result_markdown,
        "password_hash": doc.owner.password_hash,  # Oops! Leaked sensitive data
    }
```

**Why it's wrong**: No `response_model` means no output validation, no auto-generated API docs, and risk of leaking sensitive fields. Dicts have no type checking — typos silently pass.

**What to do instead**: Define response schemas in `schemas.py`, use `response_model=` on every endpoint.

### Mutable Globals for Dependency Injection

```python
# ❌ ANTI-PATTERN — Global mutable state for "DI"
_db_engine = None

def set_db_engine(engine):
    global _db_engine
    _db_engine = engine

def get_engine():
    return _db_engine

# Called from main.py at startup
set_db_engine(create_engine(...))
```

**Why it's wrong**: Hidden dependency (not visible in function signatures), hard to test (must call `set_db_engine()` in test setup), race conditions in async, no type safety.

**What to do instead**: Use FastAPI's `Depends()` system. Pass dependencies via constructors or function parameters.

### Cross-Domain Service Calls

```python
# ❌ ANTI-PATTERN — Service importing and calling another domain's service
class AuthService:
    async def register(self, data: dict) -> User:
        user = await self._create_user(data)

        # Auth domain reaching into billing domain
        billing_service = BillingService(self.db)
        await billing_service.create_customer(user)

        # Auth domain reaching into notification domain
        notification_service = NotificationService()
        await notification_service.send_welcome_email(user)

        return user
```

**Why it's wrong**: Auth now depends on billing and notifications. Changes to billing break auth. Testing auth requires mocking billing. The dependency graph becomes a web.

**What to do instead**: Router orchestration — the router calls auth, then billing, then notifications in sequence. Each service stays independent.

---

## The Goldilocks Zone

| Concern      | Under-Engineered              | Just Right                              | Over-Engineered                            |
| ------------ | ----------------------------- | --------------------------------------- | ------------------------------------------ |
| Data Access  | Raw SQL in routers            | Services + queries.py                   | Repository + Unit of Work classes          |
| Validation   | No schemas, raw dicts         | Pydantic schemas in/out                 | Validation framework + custom rules engine |
| Events       | Direct cross-service calls    | Router orchestration + BackgroundTasks  | Full event bus + saga orchestrator         |
| DI           | Global mutables               | FastAPI Depends()                       | DI container (inject, dependency-injector) |
| Architecture | All-in-one routes             | Modules with service layer              | Hexagonal/ports-and-adapters for CRUD      |
| Testing      | No tests                      | Integration tests on services           | Unit tests with mocked repositories        |
| Caching      | No caching at all             | Redis cache-aside + HTTP Cache-Control  | Distributed cache with custom invalidation |
| Resilience   | Bare try/except + log.warning | Timeout + retry_async + BackgroundTasks | Full saga orchestrator + compensation      |
| Exceptions   | HTTPException in services     | Domain exceptions + global handler      | Exception class per field per entity       |

**The goal**: Your architecture should be the **simplest thing that keeps the codebase navigable, testable, and changeable**. Add complexity only when you feel the pain of not having it — never preemptively.

## Advanced: Imperative Mapping for Pure Domain Entities

SQLAlchemy 2.x supports **imperative mapping** — separating domain entities from ORM table definitions. This allows pure Python dataclasses as domain models:

```python
# app/modules/documents/domain.py — Pure domain entity (no ORM imports)
from dataclasses import dataclass, field
from datetime import datetime
from uuid import UUID, uuid4


@dataclass
class Document:
    user_id: UUID
    filename: str
    content_type: str
    file_size: int
    id: UUID = field(default_factory=uuid4)
    ocr_status: str = "pending"
    is_deleted: bool = False
    created_at: datetime = field(default_factory=datetime.utcnow)
```

```python
# app/modules/documents/mapping.py — ORM mapping separate from entity
from sqlalchemy import Table, Column, String, Boolean, Integer, DateTime
from sqlalchemy.dialects.postgresql import UUID as PG_UUID
from app.core.database import Base, registry
from app.modules.documents.domain import Document

documents_table = Table(
    "documents", Base.metadata,
    Column("id", PG_UUID, primary_key=True),
    Column("user_id", PG_UUID, nullable=False, index=True),
    Column("filename", String, nullable=False),
    Column("content_type", String),
    Column("file_size", Integer),
    Column("ocr_status", String, default="pending"),
    Column("is_deleted", Boolean, default=False),
    Column("created_at", DateTime),
)

registry.map_imperatively(Document, documents_table)
```

**When to use this**: When your domain has rich behavior — validation rules that span multiple fields, state transitions with invariants (e.g., an `Order` that cannot be canceled after shipping), aggregate roots that protect child entity consistency. For typical FastAPI CRUD, the standard `class Document(Base)` declarative approach is simpler and sufficient.

**When NOT to use**: If your entities are mostly data containers with logic living in services (which is the case for 90% of FastAPI projects), imperative mapping adds files and indirection without meaningful benefit. This is a tool for specific situations, not a default.

## Scaling Risks — When Pragmatic DDD Hits Its Limits

### The God Service

As a module grows, its service accumulates every business operation. Watch for these signals:

```python
# ⚠️ WARNING SIGN — service has too many responsibilities
class DocumentService:
    async def upload(self, ...): ...
    async def soft_delete(self, ...): ...
    async def list_files(self, ...): ...
    async def get_by_id(self, ...): ...
    async def bulk_import(self, ...): ...
    async def export_to_pdf(self, ...): ...
    async def reprocess_ocr(self, ...): ...
    async def merge_documents(self, ...): ...
    async def share_with_team(self, ...): ...
    async def update_permissions(self, ...): ...
    async def generate_summary(self, ...): ...
    # 500+ lines, 15+ methods, 10+ imports
```

**Mitigation**: Split by **subdomain capability**, not by arbitrary size limits:

```python
# Split into focused services when a service exceeds ~300 lines
# or handles clearly distinct business capabilities

app/modules/documents/
├── service.py            # Core CRUD: upload, get, list, delete
├── processing_service.py # OCR, reprocessing, PDF export
├── sharing_service.py    # Permissions, team sharing
└── import_service.py     # Bulk import, external sync
```

Each sub-service receives `AsyncSession` independently. The router (or `orchestration.py`) orchestrates them.

### Anemic Domain Model

This architecture deliberately uses Transaction Scripts — services contain all logic, ORM models are data containers. This is the right choice for CRUD-heavy apps, but recognize the trade-off:

```python
# Anemic — all logic in service, entity is just data
class Order(Base):
    status: Mapped[str]
    shipped_at: Mapped[datetime | None]
    canceled_at: Mapped[datetime | None]

class OrderService:
    async def cancel(self, order_id: UUID, user_id: UUID) -> None:
        order = await self._get_owned_order(order_id, user_id)
        if order.status == "shipped":
            raise OrderAlreadyShippedError(order_id)
        if order.status == "canceled":
            raise OrderAlreadyCanceledError(order_id)
        order.status = "canceled"
        order.canceled_at = datetime.utcnow()
        await self.db.flush()
```

This works, but if the same cancellation logic is needed in a worker, CLI tool, or webhook handler, it must be duplicated or the service must be imported — which couples those callers to the `AsyncSession`.

**When to evolve**: If you find the same domain logic duplicated across service + worker + webhook, consider moving invariants into the entity itself:

```python
# Rich — entity protects its own invariants
@dataclass
class Order:
    status: str
    shipped_at: datetime | None = None
    canceled_at: datetime | None = None

    def cancel(self) -> None:
        """Cancel the order. Raises if not cancellable."""
        if self.status == "shipped":
            raise OrderAlreadyShippedError(self.id)
        if self.status == "canceled":
            raise OrderAlreadyCanceledError(self.id)
        self.status = "canceled"
        self.canceled_at = datetime.utcnow()

# Now any caller (service, worker, CLI) can use:
order.cancel()  # No AsyncSession dependency for the business rule
```

This requires imperative mapping (see above) to keep the entity free of ORM imports. **Only adopt this when the pain of duplication is real, not preemptive.**

### Silent Cross-Module Coupling

Even with import rules, coupling can creep in through shared database tables:

```python
# ⚠️ Module A queries Module B's table directly
# app/modules/billing/service.py
from app.modules.documents.models import Document  # Reaches into another domain

count = await db.execute(
    select(func.count(Document.id)).where(Document.user_id == user_id)
)
```

**Mitigation**: Module B should expose a **query function** as its public API, not raw model access:

```python
# ✅ Module B exposes a query function
# app/modules/documents/queries.py
async def count_user_documents(db: AsyncSession, user_id: UUID) -> int:
    result = await db.execute(
        select(func.count(Document.id)).where(
            Document.user_id == user_id, Document.is_deleted == False,
        )
    )
    return result.scalar_one()

# Module A imports the function, not the model
from app.modules.documents.queries import count_user_documents
```

This way, if `Document`'s schema changes, only `documents/queries.py` needs updating — the billing module's import still works.

### DI Container Over-Engineering

```python
# ❌ OVER-ENGINEERED — External DI container for a FastAPI app
from dependency_injector import containers, providers
from dependency_injector.wiring import inject, Provide

class Container(containers.DeclarativeContainer):
    config = providers.Configuration()
    db_session = providers.Factory(AsyncSession)
    user_service = providers.Factory(UserService, db=db_session)
    document_service = providers.Factory(DocumentService, db=db_session)

# Wiring at startup
container = Container()
container.wire(modules=[__name__])

@router.get("/")
@inject
async def list_docs(service: DocumentService = Provide[Container.document_service]):
    ...
```

**Why it's wrong**: FastAPI's `Depends()` already provides composable, testable dependency injection with zero setup. External DI containers add configuration files, wiring ceremony, and a learning curve — all for capabilities that `Depends()` handles natively.

**What to do instead**: Use `Depends()` with type aliases:

```python
# ✅ FastAPI's native DI — covers 95% of cases
# Define aliases once in shared/types.py, import everywhere
from app.shared.types import DB, CurrentUser

@router.get("/")
async def list_docs(db: DB, user: CurrentUser):
    service = DocumentService(db)
    return await service.list_files(user_id=user.id)
```

**When external DI IS justified**: Only when you have 20+ dependencies with complex lifecycle management (scoped, singleton, transient) that `Depends()` cannot express. This is extremely rare in FastAPI projects.

### Anemic Domain Model — When to Evolve

The pragmatic DDD style uses **Transaction Scripts** — all logic in services, entities as data containers. This is a conscious trade-off, not an accident. But monitor these signals:

**Signal 1: Duplicated state checks** — The same validation appears in 3+ places:

```python
# ⚠️ Same check in service, worker, and webhook handler
# service.py
if order.status == "shipped":
    raise OrderAlreadyShippedError(order.id)

# worker.py
if order.status == "shipped":
    logger.warning("Cannot process shipped order")
    return

# webhook_handler.py
if order.status == "shipped":
    raise HTTPException(400, "Order already shipped")
```

**Signal 2: Complex state transitions** — An entity has 4+ statuses with rules about valid transitions.

**Signal 3: Cross-field invariants** — A rule depends on multiple fields: "cannot set `shipped_at` unless `status == 'processing'` AND `payment_status == 'paid'`".

**What to do**: Move the invariant into the entity itself using a pure dataclass + imperative mapping:

```python
# ✅ Rich entity — protects its own invariants
@dataclass
class Order:
    status: str
    shipped_at: datetime | None = None
    payment_status: str = "pending"

    def cancel(self) -> None:
        if self.status == "shipped":
            raise OrderAlreadyShippedError(self.id)
        self.status = "canceled"

    def ship(self) -> None:
        if self.payment_status != "paid":
            raise PaymentRequiredError(self.id)
        self.status = "shipped"
        self.shipped_at = datetime.utcnow()

# Now any caller (service, worker, webhook) uses:
order.cancel()  # No duplication, no AsyncSession dependency
```

**Rule**: Only adopt rich entities when the pain of duplication is real — never preemptively. For most CRUD modules, anemic entities in `models.py` are the right choice.
