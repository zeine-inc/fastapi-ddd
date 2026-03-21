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

## ORM Registry & Migrations

### Why a Registry Is Needed

SQLAlchemy registers ORM models in `Base.metadata` **when the Python module is imported** — not before. If a module is never imported, `Base.metadata` has no knowledge of its tables.

This creates a technical requirement: **all ORM models must be imported in a central location** so that:

1. **Alembic autogenerate** detects all tables when running `alembic revision --autogenerate`
2. **Workers** (separate processes that don't import routers or services) have access to all models via the SQLAlchemy mapper

Without this, Alembic may generate migrations that **drop tables** it doesn't know about, and workers will fail with mapper errors.

### `app/orm.py` — The ORM Registry

The registry lives at `app/orm.py` — a composition-level file next to `main.py`. Just as `main.py` imports all routers to register them with FastAPI, `orm.py` imports all models to register them in `Base.metadata`.

**Why `app/` root level**: The skill's import rules prohibit `core/ → modules/*` and `shared/ → modules/*`. The registry must import from all modules, so it belongs at the composition level — not in `core/` or `shared/`.

**Why `orm.py`** (not `models.py` or `db.py`): "Models" is ambiguous in projects that also involve AI/LLM (where `models/` commonly refers to AI models). `db.py` overlaps with `core/database.py`. "ORM" is unambiguous — it means Object-Relational Mapping, universally understood by Python backend developers.

```
app/
├── main.py                    # Composition root — registers routers
├── orm.py                     # ORM registry — registers all module models
├── core/
│   └── database.py            # Engine, sessionmaker, Base, get_db()
├── modules/
│   ├── auth/models.py         # Defines User, Login
│   ├── documents/models.py    # Defines Document
│   └── billing/models.py      # Defines Subscription, Invoice
├── alembic/
│   ├── versions/
│   ├── env.py                 # Imports app.orm → target_metadata = Base.metadata
│   └── script.py.mako
└── alembic.ini
```

### Approach 1: Explicit Imports (Default)

The recommended approach — explicit, type-safe, and easy to debug:

```python
# app/orm.py — ORM Registry
"""
Centralized import point that ensures all ORM models are registered
in Base.metadata. SQLAlchemy only knows about a model after its Python
module is imported — this file guarantees every model is loaded.

Models are DEFINED in app/modules/{module}/models.py.
This file only IMPORTS them so Base.metadata is fully populated.

Referenced by:
- alembic/env.py → for autogenerate to detect all tables
- workers/*.py → for the SQLAlchemy mapper to know all entities
"""
from app.core.database import Base  # noqa: F401

# ── Auth ──
from app.modules.auth.models import User, Login  # noqa: F401

# ── Documents ──
from app.modules.documents.models import Document  # noqa: F401

# ── Billing ──
from app.modules.billing.models import Subscription, Invoice  # noqa: F401

# Adding a new module with ORM models?
# 1. Create app/modules/{module}/models.py (classes inheriting from Base)
# 2. Add the import HERE
# 3. Run: alembic revision --autogenerate -m "add {module} tables"
```

**When to use**: Most projects (< 15 modules). Full IDE support, type-checker compatible, zero magic.

### Approach 2: Auto-Discovery (Alternative)

For projects with many modules where maintaining explicit imports becomes friction:

```python
# app/orm.py — ORM Registry (Auto-Discovery)
"""
Auto-discovers and imports all ORM models from app/modules/*/models.py.
Ensures Base.metadata is fully populated for Alembic and workers.
"""
import importlib
import logging
from pathlib import Path

from app.core.database import Base  # noqa: F401

logger = logging.getLogger(__name__)


def _discover_models() -> None:
    """Import all modules/*/models.py to register ORM models in Base.metadata."""
    modules_dir = Path(__file__).parent / "modules"

    for models_file in sorted(modules_dir.glob("*/models.py")):
        module_name = models_file.parent.name
        import_path = f"app.modules.{module_name}.models"
        try:
            importlib.import_module(import_path)
            logger.debug("Loaded ORM models from %s", import_path)
        except Exception:
            logger.exception("Failed to load ORM models from %s", import_path)
            raise  # Fail fast — a missing model breaks migrations


_discover_models()
```

**When to use**: Projects with 15+ modules, teams that prefer zero-maintenance registry, standardized module structure.

**Trade-offs vs explicit imports**:

| Aspect | Explicit Imports | Auto-Discovery |
| --- | --- | --- |
| IDE / type-checker support | Full | None (dynamic imports) |
| Debuggability | Excellent | Good (with logging) |
| Maintenance on new module | Update 1 file | Zero |
| Risk of forgetting import | Exists | Eliminated |
| Circular import risk | Low (full control) | Medium |
| Scalability (20+ modules) | Works, file grows | Scales naturally |

**Design choices in the auto-discovery variant**:

- `raise` in the except block = **fail fast**. If a model fails to load, the process stops immediately. This is intentional — it is better to crash on startup than to generate a migration that drops a table.
- `sorted()` = **deterministic order** (alphabetical by module name).
- `logger.debug` on success, `logger.exception` on failure — debugging without polluting normal logs.
- **Why `pathlib.glob()` and not `pkgutil.walk_packages()`**: `pkgutil` is recursive and imports everything inside the package (services, routers, etc.) — unnecessary imports with potential side effects. `pkgutil` also has a known bug where `SystemExit` is not caught by the `onerror` callback (CPython issue #103288). `pathlib.glob("*/models.py")` imports only model files — precise and controlled.

### Consumers — Alembic & Workers

Regardless of which approach you choose, consumers use the same pattern — `import app.orm`:

```python
# alembic/env.py
import app.orm  # noqa: F401 — registers all ORM models in Base.metadata
from app.core.database import Base

target_metadata = Base.metadata
```

```python
# workers/process_task.py
import app.orm  # noqa: F401 — registers all ORM models in mapper

from app.modules.documents.models import Document
from app.core.database import get_sync_session
```

### New Module Checklist

When creating a new module with ORM models:

1. Create `app/modules/{module}/models.py` — define classes inheriting from `Base`
2. **If using explicit imports**: add the import to `app/orm.py`
3. **If using auto-discovery**: nothing to do — it is picked up automatically
4. Run: `alembic revision --autogenerate -m "add {module} tables"`
5. Review the generated migration script in `alembic/versions/`
6. Apply: `alembic upgrade head`

**Migration workflow**: `alembic revision --autogenerate -m "add documents table"` → review generated script → `alembic upgrade head`.

## Naming Conventions

| Concept        | File           | Class/Function Name                                       |
| -------------- | -------------- | --------------------------------------------------------- |
| HTTP endpoints | `router.py`    | `router = APIRouter(prefix="/things", tags=["things"])`   |
| Business logic | `service.py`   | `class ThingService:` or standalone `async def` functions |
| ORM entity     | `models.py`    | `class Thing(Base):`                                      |
| ORM registry   | `orm.py`       | Imports all models — lives at `app/orm.py` (composition)  |
| Request DTO    | `schemas.py`   | `class ThingCreateRequest(BaseModel):`                    |
| Response DTO   | `schemas.py`   | `class ThingResponse(BaseModel):`                         |
| SQL queries    | `queries.py`   | `async def count_things_this_month(db, user_id):`         |
| Guard          | `guards.py`    | `async def require_thing_access(current_user):`           |
| Constants      | `constants.py` | `THING_LIMITS = {...}` or `class ThingStatus(str, Enum):` |
