# Guards & Dependencies

## What Are Guards?

Guards are **FastAPI dependencies that enforce access rules and return the validated entity**. They act as gatekeepers — if the check fails, the request is rejected before reaching the service layer. If it passes, the guard returns the validated entity for use in the handler.

Common guard types:

- **Authentication**: Is the user logged in?
- **Authorization**: Can this user access this resource?
- **Plan enforcement**: Has the user exceeded their plan limits?
- **Rate limiting**: Too many requests?
- **Feature flags**: Is this feature enabled for this user?

## Guards vs Dependencies vs Inline — When to Use Each

The FastAPI ecosystem uses `Depends()` for everything. The skill distinguishes **guards** from other dependencies by their intent:

| Type | Intent | Returns | Lives in | Example |
| --- | --- | --- | --- | --- |
| **Guard** | Validates access, enforces rules | The validated entity | `guards.py` | `require_active_plan → User` |
| **DI type alias** | Shorthand for injecting a dependency | The resolved dependency | `shared/types.py` | `DB = Annotated[AsyncSession, Depends(get_db)]` |
| **Service factory** | Wires a service with its dependencies | The service instance | Inline in router | `service = DocumentService(db)` |

### The Boundary Rule

> **Guard** = validates + returns entity. The handler **uses** the returned value.
> **Service factory** = wires + returns service. Keep inline unless reused by 3+ routers.

### Concrete Examples

```python
# ✅ GUARD — validates access, returns validated entity
# Lives in: modules/workspaces/guards.py
async def require_workspace_access(
    workspace_id: UUID,
    current_user: CurrentUser,
    db: DB,
) -> Workspace:
    """Validate user has access to this workspace, return it."""
    workspace = await db.get(Workspace, workspace_id)
    if not workspace or current_user.id not in workspace.member_ids:
        raise HTTPException(status_code=403, detail="No access to workspace")
    return workspace

# ✅ GUARD — validates plan status, returns validated user
# Lives in: modules/subscriptions/guards.py
async def require_active_plan(current_user: CurrentUser, db: DB) -> User:
    if current_user.plan_status in ("canceled", "expired"):
        raise HTTPException(status_code=402, detail="Plan inactive")
    return current_user

# ✅ SERVICE FACTORY — wiring only, no validation
# Lives in: INLINE in router (not in a separate file)
@router.post("/")
async def create_document(file: UploadFile, db: DB, current_user: CurrentUser):
    service = DocumentService(db)  # ← inline, no file needed
    return await service.upload(file, user_id=current_user.id)

# ✅ DI TYPE ALIAS — shorthand for dependency injection
# Lives in: shared/types.py
DB = Annotated[AsyncSession, Depends(get_db)]
CurrentUser = Annotated[User, Depends(get_current_user)]
```

### When to Create `dependencies.py` in a Module

**Almost never.** Service instantiation is a one-liner in the router. Only create `dependencies.py` when:

- A module has **3+ dependencies** that are reused by other modules (not just by its own routers)
- The dependency involves non-trivial wiring (multiple infrastructure clients, feature flags)

```python
# modules/ai/dependencies.py — justified: complex wiring reused by other modules
async def get_ai_service(db: DB, settings: Settings = Depends(get_settings)) -> AIService:
    return AIService(db=db, api_key=settings.openai_key, model=settings.default_model)

AIServiceDep = Annotated[AIService, Depends(get_ai_service)]
```

### `get_current_user` — Guard or Dependency?

`get_current_user` is **infrastructure**, not a guard. It lives in `core/security.py` because:

- It's the foundational auth mechanism (JWT verification → User resolution)
- Every module needs it
- It doesn't enforce domain rules — it establishes identity

The **type alias** `CurrentUser = Annotated[User, Depends(get_current_user)]` lives in `shared/types.py`. Module-specific guards like `require_active_plan` **depend on** `CurrentUser` and add domain enforcement on top.

## Guard Pattern

A guard is a function that either **returns the validated entity** or **raises an HTTPException**:

```python
# app/modules/subscriptions/guards.py
from typing import Annotated
from fastapi import Depends, HTTPException, status

from app.shared.types import DB, CurrentUser
from app.modules.users.models import User
from app.modules.subscriptions.constants import PLAN_LIMITS
from app.modules.subscriptions.queries import count_docs_this_month


async def require_active_plan(
    current_user: CurrentUser,
    db: DB,
) -> User:
    """Reject users with expired trials or canceled plans."""
    if current_user.plan_status in ("canceled", "expired"):
        raise HTTPException(
            status_code=status.HTTP_402_PAYMENT_REQUIRED,
            detail={
                "code": "plan_inactive",
                "message": "Your plan is no longer active",
            },
        )
    return current_user


async def check_doc_limit(
    current_user: Annotated[User, Depends(require_active_plan)],
    db: DB,
) -> User:
    """Enforce monthly document upload limit based on plan."""
    limits = PLAN_LIMITS[current_user.plan]
    current_count = await count_docs_this_month(db, current_user.id)

    if current_count >= limits["docs_per_month"]:
        raise HTTPException(
            status_code=status.HTTP_402_PAYMENT_REQUIRED,
            detail={
                "code": "doc_limit_reached",
                "message": f"Monthly limit of {limits['docs_per_month']} documents reached",
                "limit": limits["docs_per_month"],
                "current": current_count,
            },
        )
    return current_user
```

## Guard Composition (Chaining)

Guards can depend on other guards, forming a chain:

```python
# require_active_plan runs FIRST
# ↓ then check_doc_limit runs (receives the validated user)
# ↓ then check_storage_limit runs

async def require_active_plan(current_user: CurrentUser, db: DB) -> User: ...

async def check_doc_limit(
    current_user: Annotated[User, Depends(require_active_plan)],  # Chains!
    db: DB,
) -> User: ...

async def check_storage_limit(
    current_user: Annotated[User, Depends(require_active_plan)],  # Also chains from active plan
    db: DB,
) -> User: ...
```

Use chaining in the **router** to apply multiple guards:

```python
@router.post("/")
async def upload_document(
    file: UploadFile,
    db: DB,
    current_user: Annotated[User, Depends(check_doc_limit)],      # Includes require_active_plan
    _storage: Annotated[User, Depends(check_storage_limit)],      # Also checks storage
):
    service = DocumentService(db)
    return await service.upload(file, user_id=current_user.id)
```

## Type Aliases for Clean DI

Use `Annotated` type aliases to keep endpoint signatures readable. Aliases live in **two places**:

**Core aliases** — in `shared/types.py` (used by every module):

```python
# app/shared/types.py
from typing import Annotated
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession

from app.core.database import get_db
from app.core.security import get_current_user
from app.modules.users.models import User

DB = Annotated[AsyncSession, Depends(get_db)]
CurrentUser = Annotated[User, Depends(get_current_user)]
```

**Guard aliases** — in the module's `guards.py` (domain-specific):

```python
# app/modules/subscriptions/guards.py
from typing import Annotated
from fastapi import Depends

ActiveUser = Annotated[User, Depends(require_active_plan)]
DocLimitedUser = Annotated[User, Depends(check_doc_limit)]
```

```python
# Clean endpoint signature
from app.shared.types import DB
from app.modules.subscriptions.guards import DocLimitedUser

@router.post("/")
async def upload(file: UploadFile, db: DB, user: DocLimitedUser):
    ...
```

## Where Guards Live

Guards belong in the **module that owns the business logic** they enforce:

| Guard                    | Module                            | Why                                  |
| ------------------------ | --------------------------------- | ------------------------------------ |
| `get_current_user`       | `core/security.py`                | Cross-cutting auth infrastructure    |
| `require_active_plan`    | `modules/subscriptions/guards.py` | Subscription domain logic            |
| `check_doc_limit`        | `modules/subscriptions/guards.py` | Plan limits are billing concepts     |
| `require_admin`          | `core/security.py`                | Role-based access is cross-cutting   |
| `require_document_owner` | `modules/documents/guards.py`     | Document ownership is document logic |

Other modules **import and use** these guards — that's the public interface of the module:

```python
# In documents/router.py
from app.shared.types import DB
from app.modules.subscriptions.guards import DocLimitedUser

@router.post("/")
async def upload(file: UploadFile, db: DB, user: DocLimitedUser):
    service = DocumentService(db)
    return await service.upload(file, user_id=user.id)
```

This is NOT coupling — it's consuming a module's public API. The documents module doesn't know HOW plan limits work, it just applies the guard.

## Parameterized Guards

For guards that need configuration, use a factory function:

```python
# app/modules/subscriptions/guards.py

def require_model_access(model_id: str):
    """Factory: returns a guard that checks if user's plan allows this model."""
    async def guard(
        current_user: Annotated[User, Depends(require_active_plan)],
    ) -> User:
        limits = PLAN_LIMITS[current_user.plan]
        if model_id not in limits["allowed_models"]:
            raise HTTPException(
                status_code=status.HTTP_402_PAYMENT_REQUIRED,
                detail={
                    "code": "model_not_available",
                    "message": f"Model {model_id} requires a higher plan",
                },
            )
        return current_user
    return guard

# Usage in router:
@router.post("/chat")
async def chat(
    user: Annotated[User, Depends(require_model_access("gpt-4"))],
):
    ...
```

## Guard vs Service Validation

| Check                        | Where                   | Why                                  |
| ---------------------------- | ----------------------- | ------------------------------------ |
| Is user authenticated?       | Guard                   | Reusable, runs before handler        |
| Is plan active?              | Guard                   | Reusable across modules              |
| Does user own this resource? | Service                 | Needs to query the specific resource |
| Is file type valid?          | Service (via validator) | Domain-specific logic                |
| Is email unique?             | Service                 | Needs DB query in business context   |

**Rule of thumb**: If the check is **reusable across endpoints** and depends only on the user/request metadata, make it a guard. If it depends on **the specific resource being operated on**, keep it in the service.

## Error Response Convention

Use structured JSON errors with a machine-readable code:

```python
raise HTTPException(
    status_code=status.HTTP_402_PAYMENT_REQUIRED,
    detail={
        "code": "doc_limit_reached",        # Machine-readable
        "message": "Monthly limit reached",  # Human-readable
        "limit": 50,                         # Context
        "current": 50,                       # Context
    },
)
```

This allows frontend apps to show specific UI (upgrade prompts, limit indicators) based on the error code.
