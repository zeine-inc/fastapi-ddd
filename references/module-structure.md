# Module Structure & Project Layout

## The Two Zones: `core/` vs `modules/`

Every FastAPI DDD project has two distinct zones:

### `core/` — Infrastructure & Cross-Cutting Concerns

Technical plumbing that **every module might need** but that contains **no business logic and no domain contracts**:

```
app/core/
├── config.py           # Pydantic Settings — environment variables, secrets
├── database.py         # Async SQLAlchemy engine, sessionmaker, Base, get_db()
├── security.py         # JWT creation/verification, get_current_user dependency
├── storage.py          # File storage client (S3, MinIO, local)
├── cache.py            # Redis/cache client + cache_get/cache_set helpers
├── http_client.py      # Shared httpx.AsyncClient with timeout defaults
├── resilience.py       # retry_async(), CircuitBreaker — external call protection
├── pubsub.py           # Event publishing infrastructure
└── middleware.py       # Custom middleware (CORS, rate limiting, JWT)
```

**Rule**: `core/` = "how we connect". If you can describe it as technical plumbing without mentioning a business domain, it belongs in `core/`.

### `shared/` — Cross-Module Domain Contracts

Reusable domain contracts that **multiple modules need** but that belong to **no single module**:

```
app/shared/
├── schemas.py          # PaginatedResponse[T], CursorResponse[T], ErrorResponse
├── exceptions.py       # DomainError, NotFoundError, LimitExceededError, etc.
├── types.py            # DI type aliases: DB, CurrentUser
└── mixins.py           # TimestampMixin, SoftDeleteMixin (SQLAlchemy model mixins)
```

**Rule**: `shared/` = "contracts everyone uses". Generic schemas, exceptions, type aliases, and ORM mixins that are consumed by 2+ modules.

### `core/` vs `shared/` — How to Decide

| Question | `core/` | `shared/` |
| --- | --- | --- |
| Is it technical plumbing (engine, client, config)? | Yes | — |
| Is it a domain contract (schema, exception, type)? | — | Yes |
| Can you describe it without mentioning FastAPI or SQLAlchemy? | Maybe shared | — |
| Does it wrap a third-party library? | Yes | — |
| Is it a generic response shape used by multiple modules? | — | Yes |

Examples:

- "Creates JWT tokens" → `core/security.py` (infrastructure)
- "Validates uploaded file types for documents" → `modules/documents/validators.py` (domain-specific)
- "Connects to Redis" → `core/cache.py` (infrastructure)
- "Checks if user exceeded their plan's upload limit" → `modules/subscriptions/guards.py` (domain-specific)
- "Paginated response with total count" → `shared/schemas.py` (generic contract)
- "NotFoundError with entity name" → `shared/exceptions.py` (generic contract)
- "`DB = Annotated[AsyncSession, Depends(get_db)]`" → `shared/types.py` (DI alias)
- "`TimestampMixin` with `created_at`/`updated_at`" → `shared/mixins.py` (ORM contract)

### `shared/` Contents

#### `shared/schemas.py` — Generic Response Shapes

```python
# app/shared/schemas.py
from pydantic import BaseModel, Field


class PaginatedResponse[T](BaseModel):
    """Offset-based pagination wrapper — used by list endpoints across modules."""
    items: list[T]
    total: int
    page: int
    page_size: int
    pages: int


class CursorResponse[T](BaseModel):
    """Cursor-based pagination — used by feed/scroll endpoints."""
    items: list[T]
    next_cursor: str | None = None
    has_more: bool
```

Create `shared/schemas.py` when 2+ modules need the same generic response shape. Don't create it preemptively — inline schemas in the module until duplication appears.

#### `shared/exceptions.py` — Domain Exception Hierarchy

```python
# app/shared/exceptions.py
class DomainError(Exception):
    """Base class for domain errors."""
    def __init__(self, message: str, code: str):
        self.message = message
        self.code = code
        super().__init__(message)

class NotFoundError(DomainError):
    def __init__(self, entity: str, id: str | None = None):
        detail = f"{entity} not found" if not id else f"{entity} {id} not found"
        super().__init__(detail, code=f"{entity.lower()}_not_found")

class PermissionDeniedError(DomainError):
    def __init__(self, message: str = "Permission denied"):
        super().__init__(message, code="permission_denied")

class LimitExceededError(DomainError):
    def __init__(self, message: str, *, limit: int, current: int):
        self.limit = limit
        self.current = current
        super().__init__(message, code="limit_exceeded")

class ConflictError(DomainError):
    def __init__(self, message: str = "Resource already exists"):
        super().__init__(message, code="conflict")

class ServiceUnavailableError(DomainError):
    def __init__(self, message: str = "External service temporarily unavailable"):
        super().__init__(message, code="service_unavailable")
```

#### `shared/types.py` — DI Type Aliases

```python
# app/shared/types.py
from typing import Annotated
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession

from app.core.database import get_db
from app.core.security import get_current_user
from app.modules.users.models import User

# Core DI aliases — import these in routers instead of repeating Annotated[...]
DB = Annotated[AsyncSession, Depends(get_db)]
CurrentUser = Annotated[User, Depends(get_current_user)]
```

Modules import from `shared/types.py`:

```python
# app/modules/documents/router.py
from app.shared.types import DB, CurrentUser
```

Module-specific guard aliases (e.g., `ActiveUser`, `DocLimitedUser`) stay in the module's `guards.py` — they are domain-specific, not shared.

#### `shared/mixins.py` — ORM Model Mixins

```python
# app/shared/mixins.py
from datetime import datetime
from sqlalchemy.orm import Mapped, mapped_column


class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)
    updated_at: Mapped[datetime | None] = mapped_column(default=None, onupdate=datetime.utcnow)


class SoftDeleteMixin:
    is_deleted: Mapped[bool] = mapped_column(default=False, index=True)
```

### When to Add Files to `shared/`

| You need... | Add to `shared/` |
| --- | --- |
| A generic response shape used by 2+ modules | `schemas.py` |
| Domain exceptions raised from services | `exceptions.py` (create early — almost always needed) |
| DI type aliases (`DB`, `CurrentUser`) | `types.py` (create early — used by every router) |
| ORM mixins used by 2+ models | `mixins.py` |

**Do NOT put in `shared/`**: Module-specific schemas, module-specific guards, business logic, or anything that mentions a specific domain concept.

### `modules/` — Domain Bounded Contexts

Each module is a **bounded context** — a self-contained business capability:

```
app/modules/
├── auth/               # Authentication & identity
├── documents/          # Document management
├── billing/            # Payments & subscriptions
├── notifications/      # Email, push, in-app alerts
└── analytics/          # Usage tracking & reporting
```

**Rule**: Group by **business capability**, not by technical layer. Never create `app/routers/`, `app/services/`, `app/models/` as top-level directories.

## Module Anatomy

### Full Module (Complex Domain)

When a module has real business logic, validation, and multiple operations:

```
app/modules/documents/
├── __init__.py         # Empty or re-exports
├── router.py           # HTTP endpoints — thin delegation layer
├── schemas.py          # Pydantic V2 request/response DTOs
├── service.py          # Business logic — orchestrates operations
├── models.py           # SQLAlchemy ORM entities
├── queries.py          # (optional) Named query functions — add when queries are complex or reused
├── guards.py           # Domain-specific enforcement dependencies
├── constants.py        # Enums, limits, magic values
├── validators.py       # Domain-specific input validation
├── orchestration.py    # (optional) Cross-module orchestration — when router coordinates > 3 services
├── events.py           # Event publishing helpers
└── indexing.py         # Search/vector index operations
```

### Minimal Module (Shared Entity)

When a module is just a data definition consumed by other modules:

```
app/modules/users/
├── __init__.py
└── models.py           # User ORM model — CRUD lives in auth module
```

This is valid. Not every entity needs its own service. The `User` model is a shared entity — auth creates it, billing reads it, documents reference it. User management service/router only appear when you add profile editing, avatar upload, etc.

### Integration Module (Thin Wrapper)

When a module overrides or wraps an external framework's default behavior:

```
app/modules/agents/
├── __init__.py
└── router.py           # Custom route that overrides framework defaults
```

No service needed if the router just injects security filters and delegates to the framework. Add a service when custom business logic appears.

## When to Add Files to a Module

| You need to...                          | Add this file              |
| --------------------------------------- | -------------------------- |
| Handle HTTP requests                    | `router.py` (always first) |
| Validate request/response shapes        | `schemas.py`               |
| Orchestrate business logic              | `service.py`               |
| Define database tables                  | `models.py`                |
| Write complex or reused queries         | `queries.py`               |
| Enforce access rules or limits          | `guards.py`                |
| Store domain constants or enums         | `constants.py`             |
| Validate domain-specific inputs         | `validators.py`            |
| Orchestrate > 3 services across modules | `orchestration.py`         |
| Publish events for other modules        | `events.py`                |

**Start minimal, add files as complexity demands.** A module with just `router.py` is fine. A module with just `models.py` is fine.

## `main.py` as Composition Root

`main.py` should do **wiring only** — no business logic, no query logic, no complex configuration:

```python
# app/main.py — Composition Root
from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.responses import JSONResponse

from app.core.config import get_settings
from app.core.database import engine
from app.shared.exceptions import (
    NotFoundError, PermissionDeniedError, LimitExceededError, ConflictError, ServiceUnavailableError,
)
from app.modules.auth.router import router as auth_router
from app.modules.documents.router import router as documents_router
from app.modules.billing.router import router as billing_router


@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    yield
    # Shutdown
    await engine.dispose()


def create_app() -> FastAPI:
    settings = get_settings()
    app = FastAPI(title="MyApp", lifespan=lifespan)

    # Global exception handlers — translate domain exceptions to HTTP responses
    @app.exception_handler(NotFoundError)
    async def not_found_handler(request, exc: NotFoundError):
        return JSONResponse(status_code=404, content={"code": exc.code, "message": exc.message})

    @app.exception_handler(PermissionDeniedError)
    async def permission_handler(request, exc: PermissionDeniedError):
        return JSONResponse(status_code=403, content={"code": exc.code, "message": exc.message})

    @app.exception_handler(LimitExceededError)
    async def limit_handler(request, exc: LimitExceededError):
        return JSONResponse(status_code=402, content={"code": exc.code, "message": exc.message, "limit": exc.limit, "current": exc.current})

    @app.exception_handler(ConflictError)
    async def conflict_handler(request, exc: ConflictError):
        return JSONResponse(status_code=409, content={"code": exc.code, "message": exc.message})

    @app.exception_handler(ServiceUnavailableError)
    async def service_unavailable_handler(request, exc: ServiceUnavailableError):
        return JSONResponse(status_code=503, content={"code": exc.code, "message": exc.message})

    # Middleware
    app.add_middleware(...)

    # Routers
    app.include_router(auth_router)
    app.include_router(documents_router)
    app.include_router(billing_router)

    return app

app = create_app()
```

If agent/AI framework setup, registry configuration, or complex service wiring grows beyond ~20 lines, extract to `core/agent.py` or `core/setup.py` and call it from `create_app()`.

## Module Boundaries — Import Rules

### Allowed Imports

```
modules/auth/       → core/*              ✅ (infrastructure)
modules/auth/       → shared/*            ✅ (shared contracts)
modules/auth/       → modules/users/models ✅ (shared entity)
modules/documents/  → core/*              ✅
modules/documents/  → shared/*            ✅
shared/             → core/*              ✅ (shared depends on infra)
shared/types        → modules/users/models ✅ (CurrentUser needs User — accepted exception)
```

### Allowed Cross-Module Imports (Public Interface Only)

```
modules/documents/router → modules/billing/guards    ✅ (consuming public guard)
modules/auth/router      → modules/billing/service   ✅ (orchestration at router level)
```

### Forbidden Imports

```
modules/auth/service → modules/billing/service  ❌ (service-to-service coupling)
modules/documents/   → modules/auth/service     ❌ (reach into another domain's internals)
core/                → modules/*                ❌ (core must not depend on modules)
core/                → shared/*                 ❌ (core must not depend on shared)
shared/              → modules/* (except User)  ❌ (shared must not depend on domain modules)
```

**Key rule**: Services should NOT call other module's services directly. If module A needs something from module B, either:

1. The **router** orchestrates both services (router is the coordinator)
2. Module B exposes a **guard/dependency** that module A consumes
3. Module B publishes an **event** that module A subscribes to

## Protocol Contracts for Cross-Module Boundaries

When modules expose guards or functions consumed by other modules, define explicit contracts using `typing.Protocol`:

```python
# app/modules/subscriptions/protocols.py
from typing import Protocol
from uuid import UUID
from app.modules.users.models import User


class PlanEnforcer(Protocol):
    """Contract for plan enforcement guards consumed by other modules."""
    async def __call__(self, current_user: User, db: ...) -> User: ...
```

This is optional for small projects (< 5 modules). But when multiple modules import guards or query functions from another module, a Protocol contract makes the boundary explicit and protects against breaking changes.

**Practical approach**: You don't need `protocols.py` on day one. Add it when:

- 3+ modules consume the same guard or function from another module
- A module's public API changes break consumers unexpectedly
- You want IDE/mypy verification that consumers and providers agree on signatures

```python
# Instead of full Protocol files, a simpler approach: document the public API
# app/modules/subscriptions/__init__.py
"""
Public API (consumed by other modules):
- guards.require_active_plan — Guard that validates plan is active
- guards.check_doc_limit — Guard that enforces monthly document limit
- queries.count_docs_this_month — Count documents in current billing cycle
"""
```

## Migrations with Alembic

Every project using SQLAlchemy ORM should include Alembic for schema migrations:

```
app/
├── alembic/
│   ├── versions/          # Migration scripts (auto-generated)
│   ├── env.py             # Migration environment — imports Base.metadata
│   └── script.py.mako     # Template for new migrations
├── alembic.ini            # Configuration (database URL, etc.)
├── core/
│   └── database.py        # Base lives here — Alembic imports it
└── modules/
    └── ...                # Models auto-discovered via Base.metadata
```

**Key setup**: Alembic's `env.py` must import `Base` from `core/database.py` and all models from `modules/*/models.py` so that `--autogenerate` detects all tables:

```python
# alembic/env.py
from app.core.database import Base

# Import all models so Base.metadata knows about them
from app.modules.auth.models import *      # noqa
from app.modules.documents.models import *  # noqa
from app.modules.billing.models import *    # noqa

target_metadata = Base.metadata
```

**Migration workflow**: `alembic revision --autogenerate -m "add documents table"` → review generated script → `alembic upgrade head`.

## Naming Conventions

| Concept        | File           | Class/Function Name                                       |
| -------------- | -------------- | --------------------------------------------------------- |
| HTTP endpoints | `router.py`    | `router = APIRouter(prefix="/things", tags=["things"])`   |
| Business logic | `service.py`   | `class ThingService:` or standalone `async def` functions |
| ORM entity     | `models.py`    | `class Thing(Base):`                                      |
| Request DTO    | `schemas.py`   | `class ThingCreateRequest(BaseModel):`                    |
| Response DTO   | `schemas.py`   | `class ThingResponse(BaseModel):`                         |
| SQL queries    | `queries.py`   | `async def count_things_this_month(db, user_id):`         |
| Guard          | `guards.py`    | `async def require_thing_access(current_user):`           |
| Constants      | `constants.py` | `THING_LIMITS = {...}` or `class ThingStatus(str, Enum):` |
