# Performance — Latency, Optimization & Trade-offs

## The Real Cost of Layers

Each translation layer introduces measurable but usually **irrelevant** overhead compared to I/O latency:

| Operation                                    | Typical Latency | % of Total |
| -------------------------------------------- | --------------- | ---------- |
| Network round-trip (client → API)            | 1-50ms          | 30-60%     |
| PostgreSQL query execution                   | 1-20ms          | 20-40%     |
| `model_validate(from_attributes)` per entity | ~0.01-0.05ms    | **< 0.1%** |
| JSON serialization of response               | ~0.1-0.5ms      | ~1%        |
| Business logic in service                    | ~0.01-1ms       | ~1-5%      |

**Bottom line**: Removing abstraction layers to improve performance is almost always the wrong optimization. Focus on I/O, not mapping overhead.

## Optimizations That Actually Matter

These are ranked by impact — address them in order:

### 1. Eager Loading (Kill N+1 Queries)

The single most common performance killer in FastAPI + SQLAlchemy:

```python
# ❌ N+1 — each doc.author triggers a separate query in the response loop
result = await db.execute(select(Document).where(Document.user_id == user_id))
documents = result.scalars().all()
# Accessing doc.author for each doc = N additional queries

# ✅ Eager loaded — single query with JOIN
from sqlalchemy.orm import joinedload, selectinload

# Many-to-one: use joinedload
result = await db.execute(
    select(Document)
    .where(Document.user_id == user_id)
    .options(joinedload(Document.author))
)
documents = result.scalars().unique().all()

# One-to-many: use selectinload (avoids cartesian explosion)
result = await db.execute(
    select(Author)
    .options(selectinload(Author.posts))
)
```

**Rule**: If your response schema has nested objects, your query **must** have matching `.options()`. No exceptions.

| Relationship                    | Strategy                                                         | Why                                          |
| ------------------------------- | ---------------------------------------------------------------- | -------------------------------------------- |
| Many-to-one (`doc.author`)      | `joinedload()`                                                   | Single LEFT JOIN, efficient                  |
| One-to-many (`author.posts`)    | `selectinload()`                                                 | Separate IN query, avoids row multiplication |
| Many-to-many                    | `selectinload()`                                                 | Same reason                                  |
| Deep nesting (`doc.author.org`) | Chain: `.options(joinedload(Doc.author).joinedload(Author.org))` | Minimal queries                              |

### 2. Connection Pool Sizing

Misconfigured pools cause timeouts under load:

```python
engine = create_async_engine(
    settings.database_url,
    pool_size=20,           # Persistent connections (default: 5)
    max_overflow=10,        # Extra connections under load
    pool_timeout=30,        # Wait for connection from pool
    pool_recycle=1800,      # Recycle after 30 min (prevents stale)
    pool_pre_ping=True,     # Verify alive before using — MANDATORY
)

async_session = async_sessionmaker(
    engine, class_=AsyncSession,
    expire_on_commit=False,  # MANDATORY for async — prevents lazy load errors
)
```

**Critical constraint**: `workers × (pool_size + max_overflow) ≤ database max_connections`

| Deployment                | `pool_size` | `max_overflow` |
| ------------------------- | ----------- | -------------- |
| Development (1 worker)    | 5           | 5              |
| Production (2-4 workers)  | 10-20       | 10             |
| High traffic (8+ workers) | 5-10        | 5-10           |

### 3. Bulk Operations

Never loop with `session.add()` for batch work:

```python
# ❌ SLOW — N round-trips
for item in items:
    db.add(Document(user_id=user_id, filename=item["filename"]))
await db.flush()

# ✅ FAST — single INSERT
await db.execute(
    insert(Document),
    [{"user_id": user_id, "filename": item["filename"]} for item in items],
)
await db.flush()
```

| Scenario            | Approach                                   |
| ------------------- | ------------------------------------------ |
| 1-5 records         | `db.add()` loop is fine                    |
| 10+ records         | `insert().values([...])`                   |
| Update filtered set | `update().where().values()`                |
| Mass soft-delete    | `update().where().values(is_deleted=True)` |

### 4. Cursor Pagination for Large Datasets

Offset-based pagination degrades at high page numbers:

```python
# ❌ SLOW at page 5000 — database still scans all preceding rows
.offset(100_000).limit(20)

# ✅ FAST at any depth — uses index
.where(Document.created_at < cursor_timestamp)
.order_by(Document.created_at.desc())
.limit(20)
```

Use offset for admin dashboards (small datasets, total count needed). Use cursor for feeds, infinite scroll, large datasets (> 100k rows).

### 5. Cache-Aside with Redis

For data read frequently but changed rarely (counts, configs, plan limits):

```python
async def get_file_counts(self, *, user_id: UUID) -> FileCountResponse:
    cache_key = f"doc_counts:{user_id}"

    cached = await cache_get(cache_key)
    if cached:
        return FileCountResponse(**cached)

    counts = await self._compute_file_counts(user_id)
    response = FileCountResponse(**counts)
    await cache_set(cache_key, response.model_dump(), ttl=300)

    return response
```

See `references/caching.md` for full patterns (HTTP headers, ETags, invalidation).

## Pydantic V2 Performance Tips

Pydantic V2's core is written in Rust, making it **4-50x faster** than V1. But there are still optimization opportunities:

### `model_validate_json()` — Skip the Dict

When receiving JSON input, `model_validate_json()` parses and validates in a single Rust pass, bypassing Python dict creation:

```python
# Standard — creates intermediate dict (2 steps)
data = json.loads(request_body)
model = MyModel.model_validate(data)

# Optimized — single Rust pass (~2x faster)
model = MyModel.model_validate_json(request_body)
```

FastAPI uses this automatically for request body parsing, but use it explicitly when processing JSON from external sources (webhooks, message queues, file imports).

### `from_attributes` — Only with ORM Objects

`from_attributes=True` adds attribute access overhead. Only enable it on response schemas that read from ORM instances:

```python
# ✅ CORRECT — reads from SQLAlchemy model attributes
class UserResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)  # Needed here
    id: UUID
    email: str

# ❌ WASTEFUL — input schemas never read from ORM objects
class UserCreateRequest(BaseModel):
    model_config = ConfigDict(from_attributes=True)  # Not needed!
    email: str
    password: str
```

### `TypeAdapter` for Lightweight Validation

When you don't need a full `BaseModel` class (e.g., validating a list of UUIDs):

```python
from pydantic import TypeAdapter

uuid_list_adapter = TypeAdapter(list[UUID])

# Lighter than creating a BaseModel wrapper
validated = uuid_list_adapter.validate_python(raw_list)
```

## Async vs Sync — When to Choose

| Aspect                           | Sync                   | Async                                            |
| -------------------------------- | ---------------------- | ------------------------------------------------ |
| **Throughput under concurrency** | Limited by thread pool | Scales via event loop                            |
| **Per-request overhead**         | Baseline               | +10-20% Python overhead                          |
| **Lazy loading**                 | Works                  | **Forbidden** — causes sync I/O in async context |
| **Connection pool**              | Simple                 | Requires careful configuration                   |
| **Celery / background workers**  | ✅ Native              | ❌ Needs separate sync session                   |

**Default**: Use async for web APIs (FastAPI's strength). Keep sync sessions for background workers (Celery).

### Mandatory Async Configuration

These two settings are **non-negotiable** in production:

1. **`pool_pre_ping=True`** — Without this, stale connections (closed by DB idle timeout, failover) cause `ConnectionError` on first use
2. **`expire_on_commit=False`** — Without this, accessing ORM attributes after commit triggers synchronous lazy load, which **fails** in async

## When Abstraction Actually Hurts

If you see these symptoms, the problem is NOT your layer count — it's a specific optimization issue:

| Symptom                                 | Root Cause                          | Solution                                   |
| --------------------------------------- | ----------------------------------- | ------------------------------------------ |
| Response time > 200ms on list endpoints | N+1 queries (lazy loading)          | `joinedload()` / `selectinload()`          |
| Connection pool exhausted, timeouts     | Pool undersized for worker count    | Adjust `pool_size` + use PgBouncer         |
| Memory spikes during bulk import        | Loop with `session.add()`           | `insert().values([...])`                   |
| Slow analytics/reporting endpoints      | Heavy analytical queries on OLTP DB | Materialized views or Redis cache          |
| High P99 latency on simple GETs         | Missing database indexes            | Add indexes on frequently filtered columns |

## Anti-Pattern: Removing Layers for "Performance"

```python
# ❌ "Optimization" — removing service layer for speed
@router.get("/documents", response_model=list[DocumentResponse])
async def list_documents(db: DB, current_user: CurrentUser):
    result = await db.execute(
        select(Document).where(
            Document.user_id == current_user.id,
            Document.is_deleted == False,
        )
    )
    return result.scalars().all()
# Saves ~0.01ms but loses: testability, reusability, domain exception handling

# ✅ Keep the service — the overhead is negligible
@router.get("/documents", response_model=DocumentListResponse)
async def list_documents(db: DB, current_user: CurrentUser):
    service = DocumentService(db)
    return await service.list_files(user_id=current_user.id)
```

The service call adds ~0.01ms overhead. A missing `joinedload()` adds ~50ms. **Optimize the right thing.**
