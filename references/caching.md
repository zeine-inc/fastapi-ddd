# Caching Patterns

Caching is the single most impactful optimization for API performance — more effective than reducing abstraction layers or optimizing serialization. Apply caching at the right level for your use case.

## When to Add Caching

Don't cache preemptively. Add caching when:

- The same query runs repeatedly with identical parameters (user dashboards, config lookups)
- An endpoint calls an expensive external service (LLM, geocoding, exchange rates)
- Response data changes infrequently relative to read frequency
- Database load becomes a bottleneck despite proper indexing

## Three Caching Levels

```
HTTP Cache Headers  →  Application Cache (Redis)  →  Query-Level Cache
   client-side           shared across requests        per-query memoization
```

## Level 1: HTTP Cache Headers

Zero-infrastructure caching — the client and CDN do the work:

```python
# app/modules/documents/router.py
from fastapi import Response

@router.get("/{doc_id}", response_model=DocumentDetailResponse)
async def get_document(
    doc_id: UUID,
    db: DB,
    current_user: CurrentUser,
    response: Response,
):
    service = DocumentService(db)
    document = await service.get_by_id(doc_id, user_id=current_user.id)

    # Cache for 60 seconds, but allow revalidation
    response.headers["Cache-Control"] = "private, max-age=60"
    # ETag for conditional requests
    response.headers["ETag"] = f'"{document.id}-{document.updated_at.isoformat()}"'

    return document
```

### Cache-Control Patterns

| Scenario                             | Header                                | Why                           |
| ------------------------------------ | ------------------------------------- | ----------------------------- |
| User-specific data                   | `private, max-age=60`                 | Only client caches, not CDN   |
| Public static data (plans, features) | `public, max-age=3600`                | CDN + client cache            |
| Frequently changing data             | `private, no-cache`                   | Always revalidate with server |
| Immutable assets (uploaded files)    | `public, max-age=31536000, immutable` | Cache forever                 |

### ETag / Conditional Requests

Avoid resending unchanged data:

```python
from fastapi import Header, HTTPException, status

@router.get("/{doc_id}", response_model=DocumentDetailResponse)
async def get_document(
    doc_id: UUID,
    db: DB,
    current_user: CurrentUser,
    response: Response,
    if_none_match: str | None = Header(None),
):
    service = DocumentService(db)
    document = await service.get_by_id(doc_id, user_id=current_user.id)

    etag = f'"{document.id}-{document.updated_at.isoformat()}"'
    if if_none_match == etag:
        raise HTTPException(status_code=status.HTTP_304_NOT_MODIFIED)

    response.headers["ETag"] = etag
    response.headers["Cache-Control"] = "private, max-age=0, must-revalidate"
    return document
```

## Level 2: Application Cache (Redis)

For data shared across requests or expensive to compute:

```python
# app/core/cache.py
import json
from typing import Any
from redis.asyncio import Redis

_redis: Redis | None = None


async def get_redis() -> Redis:
    global _redis
    if _redis is None:
        _redis = Redis.from_url("redis://localhost:6379", decode_responses=True)
    return _redis


async def cache_get(key: str) -> Any | None:
    """Get a cached value, returns None on miss."""
    redis = await get_redis()
    value = await redis.get(key)
    return json.loads(value) if value else None


async def cache_set(key: str, value: Any, ttl: int = 300) -> None:
    """Set a cached value with TTL in seconds."""
    redis = await get_redis()
    await redis.set(key, json.dumps(value, default=str), ex=ttl)


async def cache_delete(key: str) -> None:
    """Delete a cached value."""
    redis = await get_redis()
    await redis.delete(key)


async def cache_delete_pattern(pattern: str) -> None:
    """Delete all keys matching a pattern. Use sparingly."""
    redis = await get_redis()
    async for key in redis.scan_iter(match=pattern, count=100):
        await redis.delete(key)
```

### Cache-Aside Pattern in Services

The service checks cache first, falls back to DB, then populates cache:

```python
# app/modules/documents/service.py
from app.core.cache import cache_get, cache_set, cache_delete

class DocumentService:
    def __init__(self, db: AsyncSession):
        self.db = db

    async def get_file_counts(self, *, user_id: UUID) -> FileCountResponse:
        cache_key = f"doc_counts:{user_id}"

        # 1. Check cache
        cached = await cache_get(cache_key)
        if cached:
            return FileCountResponse(**cached)

        # 2. Query database
        counts = await self._compute_file_counts(user_id)
        response = FileCountResponse(**counts)

        # 3. Populate cache (TTL 5 minutes)
        await cache_set(cache_key, response.model_dump(), ttl=300)

        return response

    async def upload(self, file: UploadFile, *, user_id: UUID) -> DocumentResponse:
        # ... upload logic ...

        # Invalidate cached counts after mutation
        await cache_delete(f"doc_counts:{user_id}")

        return DocumentResponse.model_validate(document)
```

### Cache Key Conventions

```python
# Pattern: {module}:{entity}:{scope}:{identifier}
"doc_counts:user:{user_id}"           # User's document counts
"doc_detail:{doc_id}"                  # Single document detail
"billing:plan:{plan_name}"            # Plan configuration (shared)
"auth:user:{user_id}"                 # User profile from JWT lookup
```

**Rules**:

- Always include `user_id` in keys for user-scoped data (multi-tenancy)
- Use short, predictable prefixes for pattern-based invalidation
- Include a version suffix if the cached shape might change: `"doc_counts:v2:{user_id}"`

## Level 3: Query-Level Cache (In-Memory)

For data that changes very rarely and is read on nearly every request:

```python
# app/core/cache.py
from functools import lru_cache
from pydantic_settings import BaseSettings


@lru_cache
def get_settings() -> Settings:
    """Settings loaded once, cached for process lifetime."""
    return Settings()
```

For async data that needs periodic refresh:

```python
# app/modules/billing/service.py
from datetime import datetime, timedelta

_plan_cache: dict | None = None
_plan_cache_at: datetime | None = None
PLAN_CACHE_TTL = timedelta(minutes=30)


async def get_plan_limits(db: AsyncSession) -> dict:
    """Plan limits change rarely — cache in-memory with TTL."""
    global _plan_cache, _plan_cache_at

    now = datetime.utcnow()
    if _plan_cache and _plan_cache_at and (now - _plan_cache_at) < PLAN_CACHE_TTL:
        return _plan_cache

    result = await db.execute(select(Plan).where(Plan.is_active == True))
    plans = {p.name: p.limits for p in result.scalars().all()}

    _plan_cache = plans
    _plan_cache_at = now
    return plans
```

**When to use in-memory cache**: Only for truly global, rarely-changing data (settings, plan configs, feature flags). For user-scoped or frequently-changing data, use Redis.

## Invalidation Strategies

| Strategy             | When to Use                                    | How                                                         |
| -------------------- | ---------------------------------------------- | ----------------------------------------------------------- |
| TTL-based            | Data staleness is acceptable (counts, stats)   | `cache_set(key, value, ttl=300)`                            |
| Write-through        | Every mutation must immediately reflect        | `cache_set()` after `db.flush()` in the same service method |
| Invalidate-on-write  | Simpler than write-through, next read rebuilds | `cache_delete(key)` after mutation                          |
| Pattern invalidation | Mutation affects multiple cached entries       | `cache_delete_pattern(f"doc_*:{user_id}")`                  |

### Invalidation in Practice

```python
class DocumentService:
    async def soft_delete(self, document_id: UUID, *, user_id: UUID) -> None:
        document = await self._get_owned_document(document_id, user_id)
        document.is_deleted = True
        await self.db.flush()

        # Invalidate all caches affected by this mutation
        await cache_delete(f"doc_detail:{document_id}")
        await cache_delete(f"doc_counts:{user_id}")
```

## What NOT to Cache

- Authentication tokens or session data (use proper session stores)
- Data that must be real-time consistent (financial balances, inventory counts in checkout)
- Large blobs (use object storage with signed URLs instead)
- Data that varies per-request (pagination cursors, filtered results with many filter combinations)

## Anti-Patterns

```python
# ❌ Caching in the router — cache logic belongs in the service
@router.get("/counts")
async def get_counts(db: DB, current_user: CurrentUser):
    cached = await cache_get(f"counts:{current_user.id}")
    if cached:
        return cached
    # ...

# ✅ Router stays thin — service handles caching transparently
@router.get("/counts", response_model=FileCountResponse)
async def get_counts(db: DB, current_user: CurrentUser):
    service = DocumentService(db)
    return await service.get_file_counts(user_id=current_user.id)

# ❌ Caching without TTL — stale data forever
await redis.set(key, value)  # No expiration!

# ✅ Always set a TTL — even a long one
await redis.set(key, value, ex=3600)  # 1 hour max staleness
```
