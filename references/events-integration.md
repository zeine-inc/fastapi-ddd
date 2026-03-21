# Events & Integration

## Separating Infrastructure from Domain

Services should express **what** happens, not **how** the infrastructure delivers it. Extract infrastructure calls into helper modules:

### Before — Infrastructure in Service

```python
# ❌ Service knows about Redis, Qdrant, and their APIs
class DocumentService:
    async def soft_delete(self, doc_id: UUID, user_id: UUID) -> None:
        document = await self._get_owned_document(doc_id, user_id)
        document.is_deleted = True
        await self.db.flush()

        # Infrastructure leak: raw Qdrant API
        vector_db = get_vector_db()
        points = vector_db.client.scroll(
            collection_name=settings.qdrant_collection,
            scroll_filter=Filter(must=[
                FieldCondition(key="meta_data.document_id", match=MatchValue(value=str(doc_id)))
            ]),
        )
        for point in points[0]:
            vector_db.client.set_payload(
                collection_name=settings.qdrant_collection,
                payload={"meta_data": {**point.payload.get("meta_data", {}), "is_deleted": True}},
                points=[point.id],
            )

        # Infrastructure leak: raw Redis pub/sub
        redis = await get_redis()
        await redis.publish(
            f"documents:{user_id}",
            json.dumps({"type": "status-change", "data": {...}})
        )
```

### After — Clean Separation

```python
# ✅ Service expresses intent, helpers handle infrastructure

# app/modules/documents/service.py
from app.modules.documents.indexing import mark_document_deleted_in_index
from app.modules.documents.events import publish_document_deleted

class DocumentService:
    async def soft_delete(self, doc_id: UUID, user_id: UUID) -> None:
        document = await self._get_owned_document(doc_id, user_id)
        document.is_deleted = True
        await self.db.flush()

        await mark_document_deleted_in_index(doc_id)
        await publish_document_deleted(user_id, doc_id)
```

## Module-Level Event Helpers

### `events.py` — Domain Event Publishing

```python
# app/modules/documents/events.py
import json
from uuid import UUID
from app.core.pubsub import get_redis


async def publish_document_status_changed(
    user_id: UUID,
    document_id: UUID,
    status: str,
    filename: str,
) -> None:
    """Notify subscribers that a document's processing status changed."""
    redis = await get_redis()
    await redis.publish(
        f"documents:{user_id}",
        json.dumps({
            "type": "status-change",
            "data": {
                "document_id": str(document_id),
                "status": status,
                "filename": filename,
            },
        }),
    )


async def publish_document_counts_updated(user_id: UUID, counts: dict) -> None:
    """Notify subscribers of updated document counts."""
    redis = await get_redis()
    await redis.publish(
        f"documents:{user_id}",
        json.dumps({"type": "counts", "data": counts}),
    )


async def publish_document_deleted(user_id: UUID, document_id: UUID) -> None:
    """Notify subscribers that a document was deleted."""
    redis = await get_redis()
    await redis.publish(
        f"documents:{user_id}",
        json.dumps({
            "type": "deleted",
            "data": {"document_id": str(document_id)},
        }),
    )
```

### `indexing.py` — Search Index Operations

```python
# app/modules/documents/indexing.py
from uuid import UUID
from app.core.config import get_settings
from app.core.knowledge import get_vector_db


async def mark_document_deleted_in_index(document_id: UUID) -> None:
    """Update vector store metadata to mark document as deleted."""
    settings = get_settings()
    vector_db = get_vector_db()

    points, _ = vector_db.client.scroll(
        collection_name=settings.qdrant_collection,
        scroll_filter=Filter(must=[
            FieldCondition(
                key="meta_data.document_id",
                match=MatchValue(value=str(document_id)),
            )
        ]),
        limit=1000,
    )

    for point in points:
        meta = point.payload.get("meta_data", {})
        meta["is_deleted"] = True
        vector_db.client.set_payload(
            collection_name=settings.qdrant_collection,
            payload={"meta_data": meta},
            points=[point.id],
        )
```

## Decoupling Between Bounded Contexts

### Rule: Services Never Call Other Module's Services

```python
# ❌ Auth service calling billing service
class AuthService:
    async def register_user(self, data: dict) -> User:
        user = User(**data)
        self.db.add(user)
        await self.db.flush()

        # WRONG: crossing domain boundary at service level
        billing_service = BillingService(self.db)
        await billing_service.create_customer(user)
        return user
```

### Solution 1: Router Orchestration (Simplest)

The router is the **coordinator** — it can call multiple services:

```python
# ✅ Router orchestrates cross-domain operations
@router.post("/register", response_model=AuthResponse)
async def register(
    data: RegisterRequest,
    db: DB,
    background_tasks: BackgroundTasks,
):
    # Step 1: Auth domain (critical — must succeed)
    auth_service = AuthService(db)
    user, is_new = await auth_service.create_user(data)

    # Step 2: Billing domain (non-critical — retry in background)
    if is_new:
        background_tasks.add_task(create_stripe_customer_with_retry, user.id)

    # Step 3: Return auth tokens
    tokens = auth_service.generate_tokens(user)
    return AuthResponse(user=user, tokens=tokens)
```

> **Note**: Avoid bare `try/except Exception: logger.warning(...)` for external calls — errors get swallowed and never retried. Use `BackgroundTasks` for non-critical side effects or `retry_async()` for critical ones. See `references/resilience.md` for patterns.

### Solution 1.5: Orchestration (When Router Coordination Gets Complex)

When a router coordinates > 3 services or the orchestration logic includes error handling, retries, or compensating actions, extract it to an `orchestration.py`:

```python
# app/modules/auth/orchestration.py
from uuid import UUID
from fastapi import BackgroundTasks
from app.modules.auth.service import AuthService
from app.modules.auth.schemas import RegisterRequest, AuthResponse
from app.modules.billing.service import create_stripe_customer_with_retry
from app.modules.notifications.service import send_welcome_email

import logging
logger = logging.getLogger(__name__)


async def register_user(
    data: RegisterRequest,
    db: AsyncSession,
    background_tasks: BackgroundTasks,
) -> AuthResponse:
    """Orchestrate user registration across auth, billing, and notifications."""
    auth_service = AuthService(db)
    user, is_new = await auth_service.create_user(data)

    if is_new:
        # Non-critical side effects — run in background with retry
        background_tasks.add_task(create_stripe_customer_with_retry, user.id)
        background_tasks.add_task(send_welcome_email, user.id)

    tokens = auth_service.generate_tokens(user)
    return AuthResponse(user=user, tokens=tokens)
```

```python
# app/modules/auth/router.py — stays thin
from app.modules.auth.orchestration import register_user

@router.post("/register", response_model=AuthResponse)
async def register(data: RegisterRequest, db: DB, background_tasks: BackgroundTasks):
    return await register_user(data, db, background_tasks)
```

**When to use `orchestration.py`**:

- Router orchestrates > 3 services or modules
- Cross-module coordination includes error handling or compensating logic
- The orchestration logic is complex enough to deserve its own tests

**When NOT to use**: For 1-2 service calls, keep the orchestration in the router.

### Solution 2: Event-Driven (When Scale Demands It)

For operations that should be fire-and-forget:

```python
# Lightweight approach — background tasks
from fastapi import BackgroundTasks

@router.post("/register")
async def register(
    data: RegisterRequest,
    db: DB,
    background_tasks: BackgroundTasks,
):
    auth_service = AuthService(db)
    user, is_new = await auth_service.create_user(data)

    if is_new:
        background_tasks.add_task(create_stripe_customer, user)
        background_tasks.add_task(send_welcome_email, user)

    return AuthResponse(...)
```

## Workers as Domain Extensions

Background workers (Celery, ARQ) are part of the domain — they execute domain logic asynchronously.

> **ORM registry**: Workers run in separate processes that don't import `main.py` or routers. To ensure all ORM models are registered in SQLAlchemy's mapper, workers must `import app.orm` at the top of the module. See `references/module-structure.md` → "ORM Registry & Migrations" for details.

```python
# app/workers/ocr_task.py
import app.orm  # noqa: F401 — registers all ORM models in mapper

from app.modules.documents.models import Document
from app.modules.documents.events import (
    publish_document_status_changed,
    publish_document_counts_updated,
)
from app.modules.documents.queries import build_file_count_query

@celery_app.task(bind=True, max_retries=5, default_retry_delay=None)
def process_document(self, document_id: str):
    with get_sync_session() as session:
        document = session.get(Document, document_id)

        # Update status
        document.ocr_status = "processing"
        session.commit()
        publish_document_status_changed_sync(...)

        try:
            # Process
            result = call_ocr_service(document)
        except TransientError as exc:
            # Exponential backoff: 2s, 4s, 8s, 16s, 32s
            delay = min(2 ** self.request.retries, 300)
            raise self.retry(exc=exc, countdown=delay)
        except PermanentError:
            document.ocr_status = "failed"
            session.commit()
            return

        # Complete
        document.ocr_status = "completed"
        document.ocr_result = result
        session.commit()
        publish_document_status_changed_sync(...)
```

> See `references/resilience.md` for retry patterns, circuit breakers, and graceful degradation.

Workers can import from:

- `modules/*/models.py` — to access entities
- `modules/*/queries.py` — to run shared queries
- `modules/*/events.py` — to publish domain events
- `core/*` — to access infrastructure

Workers should NOT import from routers or services directly.

## When to Use What

| Scenario                                  | Approach                                                      |
| ----------------------------------------- | ------------------------------------------------------------- |
| Module A needs data from Module B         | Module A calls Module B's query function                      |
| Module A triggers action in Module B      | Router orchestrates both services                             |
| Fire-and-forget side effect               | `BackgroundTasks` or Celery                                   |
| Many modules react to same event          | Consider a simple event bus (but verify you actually need it) |
| External webhook triggers internal action | Webhook router → relevant service                             |

## When NOT to Build a Formal Event System

Don't build an event bus, message broker integration, or pub/sub pattern between modules when:

- You have fewer than 5 modules
- Events only flow in one direction (A → B, never B → A)
- Only 1-2 subscribers per event type
- Router orchestration is clear and maintainable

The router orchestration pattern handles 90% of cross-domain coordination needs for small-to-medium applications.
