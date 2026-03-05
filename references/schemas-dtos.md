# Schemas & DTOs — Pydantic V2

## Role of Schemas in DDD

Schemas are **Data Transfer Objects (DTOs)** — they define the shape of data crossing boundaries:

- **Request schemas**: What the client sends (input validation)
- **Response schemas**: What the API returns (output contract)
- **Internal schemas**: Data shapes between services (when dicts aren't enough)

Schemas are NOT ORM models. They are independent of your database layer.

## Schema Organization

All schemas for a module live in `schemas.py`:

```python
# app/modules/documents/schemas.py
from datetime import datetime
from uuid import UUID
from pydantic import BaseModel, Field, ConfigDict


# ---- Request DTOs ----

class DocumentUploadRequest(BaseModel):
    """Metadata sent alongside file upload."""
    description: str | None = Field(None, max_length=500)
    tags: list[str] = Field(default_factory=list, max_length=10)


# ---- Response DTOs ----

class DocumentResponse(BaseModel):
    """Single document representation."""
    model_config = ConfigDict(from_attributes=True)

    id: UUID
    filename: str
    content_type: str
    file_size: int
    ocr_status: str
    created_at: datetime
    updated_at: datetime | None = None


class DocumentDetailResponse(DocumentResponse):
    """Extended document with OCR result and download URL."""
    ocr_result_markdown: str | None = None
    file_url: str | None = None


class DocumentListResponse(BaseModel):
    """Paginated list of documents."""
    items: list[DocumentResponse]
    total: int


class FileCountResponse(BaseModel):
    """Document counts grouped by status."""
    total: int = 0
    pending: int = 0
    processing: int = 0
    completed: int = 0
    failed: int = 0
```

## Naming Convention

| Pattern                  | Use For                | Example              |
| ------------------------ | ---------------------- | -------------------- |
| `{Entity}CreateRequest`  | POST body              | `UserCreateRequest`  |
| `{Entity}UpdateRequest`  | PATCH/PUT body         | `UserUpdateRequest`  |
| `{Entity}Response`       | Single entity output   | `UserResponse`       |
| `{Entity}DetailResponse` | Extended entity output | `UserDetailResponse` |
| `{Entity}ListResponse`   | Paginated list output  | `UserListResponse`   |
| `{Entity}FilterParams`   | Query parameters       | `UserFilterParams`   |

## Key Patterns

### `from_attributes=True` — ORM to Schema

Allows automatic conversion from SQLAlchemy models:

```python
class UserResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: UUID
    email: str
    name: str
    created_at: datetime

# Usage in service:
user = await db.get(User, user_id)
return UserResponse.model_validate(user)  # Reads from ORM attributes
```

### Input vs Output Separation

Never use the same schema for both request and response:

```python
# ❌ BAD — same schema for input and output
class User(BaseModel):
    id: UUID           # Not in POST request
    email: str
    password: str      # Not in GET response
    created_at: datetime  # Not in POST request

# ✅ GOOD — separate concerns
class UserCreateRequest(BaseModel):
    email: str
    password: str

class UserResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: UUID
    email: str
    created_at: datetime
    # No password — never exposed
```

### Partial Updates with `exclude_unset`

```python
class UserUpdateRequest(BaseModel):
    name: str | None = None
    bio: str | None = None
    avatar_url: str | None = None

# In service:
async def update_user(self, user_id: UUID, data: UserUpdateRequest) -> UserResponse:
    update_data = data.model_dump(exclude_unset=True)  # Only fields the client sent
    await self.db.execute(
        update(User).where(User.id == user_id).values(**update_data)
    )
```

### Nested Responses

```python
class AuthorResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: UUID
    name: str

class PostResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: UUID
    title: str
    content: str
    author: AuthorResponse  # Nested — requires eager loading in query
```

### Validation with `field_validator`

```python
from pydantic import BaseModel, field_validator

class CheckoutRequest(BaseModel):
    plan: str
    success_url: str
    cancel_url: str

    @field_validator("plan")
    @classmethod
    def validate_plan(cls, v: str) -> str:
        valid_plans = {"starter", "professional", "enterprise"}
        if v not in valid_plans:
            raise ValueError(f"Plan must be one of: {', '.join(valid_plans)}")
        return v
```

## Using `response_model` on Endpoints

Always declare `response_model` — it provides:

1. Automatic response validation
2. Auto-generated OpenAPI documentation
3. Field filtering (strips extra fields like passwords)

```python
@router.get("/{doc_id}", response_model=DocumentDetailResponse)
async def get_document(doc_id: UUID, db: DB, current_user: CurrentUser):
    service = DocumentService(db)
    return await service.get_by_id(doc_id, user_id=current_user.id)

@router.get("/", response_model=DocumentListResponse)
async def list_documents(db: DB, current_user: CurrentUser):
    service = DocumentService(db)
    return await service.list_files(user_id=current_user.id)

@router.delete("/{doc_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_document(doc_id: UUID, db: DB, current_user: CurrentUser) -> None:
    service = DocumentService(db)
    await service.soft_delete(doc_id, user_id=current_user.id)
```

## Schemas vs ORM Models

| Aspect        | Schema (Pydantic)               | Model (SQLAlchemy)             |
| ------------- | ------------------------------- | ------------------------------ |
| Purpose       | API contract                    | Database persistence           |
| Location      | `schemas.py`                    | `models.py`                    |
| Validation    | Input validation, serialization | DB constraints                 |
| Relationships | Nested schemas (flat)           | ORM relationships (lazy/eager) |
| Identity      | No `id` on create requests      | Always has `id`                |
| Secrets       | Never includes passwords/tokens | May store hashed passwords     |

## Pagination Patterns

### Offset-Based Pagination (Simple)

Best for admin dashboards and small datasets where total count matters:

```python
class PaginationParams(BaseModel):
    page: int = Field(1, ge=1)
    page_size: int = Field(20, ge=1, le=100)

    @property
    def offset(self) -> int:
        return (self.page - 1) * self.page_size


class PaginatedResponse[T](BaseModel):
    items: list[T]
    total: int
    page: int
    page_size: int
    pages: int  # total // page_size + (1 if total % page_size else 0)


# In service:
async def list_documents(
    self, *, user_id: UUID, pagination: PaginationParams
) -> PaginatedResponse[DocumentResponse]:
    # Count total
    count_result = await self.db.execute(
        select(func.count(Document.id)).where(
            Document.user_id == user_id, Document.is_deleted == False
        )
    )
    total = count_result.scalar_one()

    # Fetch page
    result = await self.db.execute(
        select(Document)
        .where(Document.user_id == user_id, Document.is_deleted == False)
        .order_by(Document.created_at.desc())
        .offset(pagination.offset)
        .limit(pagination.page_size)
    )
    items = [DocumentResponse.model_validate(doc) for doc in result.scalars().all()]

    return PaginatedResponse(
        items=items,
        total=total,
        page=pagination.page,
        page_size=pagination.page_size,
        pages=total // pagination.page_size + (1 if total % pagination.page_size else 0),
    )
```

### Cursor-Based Pagination (Scalable)

Best for infinite scroll, mobile feeds, and large datasets:

```python
class CursorParams(BaseModel):
    cursor: datetime | None = None  # created_at of last item seen
    limit: int = Field(20, ge=1, le=100)


class CursorResponse[T](BaseModel):
    items: list[T]
    next_cursor: str | None = None  # Opaque cursor for next page
    has_more: bool


# In service:
async def list_documents_cursor(
    self, *, user_id: UUID, params: CursorParams
) -> CursorResponse[DocumentResponse]:
    query = (
        select(Document)
        .where(Document.user_id == user_id, Document.is_deleted == False)
        .order_by(Document.created_at.desc())
        .limit(params.limit + 1)  # Fetch one extra to check has_more
    )
    if params.cursor:
        query = query.where(Document.created_at < params.cursor)

    result = await self.db.execute(query)
    items = result.scalars().all()

    has_more = len(items) > params.limit
    items = items[:params.limit]

    return CursorResponse(
        items=[DocumentResponse.model_validate(doc) for doc in items],
        next_cursor=items[-1].created_at.isoformat() if has_more and items else None,
        has_more=has_more,
    )
```

### When to Use Which

| Use Case                             | Pattern | Why                                       |
| ------------------------------------ | ------- | ----------------------------------------- |
| Admin panel, table with page numbers | Offset  | Users expect page numbers and total count |
| Mobile feed, infinite scroll         | Cursor  | Consistent results as new items are added |
| Large dataset (> 100k rows)          | Cursor  | Offset becomes slow for high page numbers |
| Small dataset with filtering         | Offset  | Simpler to implement, performance is fine |

## Anti-Patterns

```python
# ❌ Returning raw dicts from services
async def list_files(self):
    return {"items": [...], "total": 5}  # No type safety, no validation

# ✅ Returning schemas
async def list_files(self) -> DocumentListResponse:
    return DocumentListResponse(items=[...], total=5)

# ❌ Using ORM model as response
@router.get("/", response_model=User)  # Exposes password_hash, internal fields
async def get_user(): ...

# ✅ Using dedicated response schema
@router.get("/", response_model=UserResponse)  # Only safe fields
async def get_user(): ...
```

## Performance: `model_validate_json()`

For JSON inputs from external sources (webhooks, message queues), use `model_validate_json()` to skip intermediate dict creation:

```python
# Standard — 2 steps (JSON parse → dict → validate)
import json
data = json.loads(webhook_body)
event = WebhookEvent.model_validate(data)

# Optimized — single Rust pass (~2x faster for large payloads)
event = WebhookEvent.model_validate_json(webhook_body)
```

FastAPI handles this automatically for request bodies, but use it explicitly when processing JSON from non-HTTP sources.

**When to use**: Webhook handlers, message queue consumers, bulk file imports, any code that parses JSON strings directly.

## The Model Triangle: When to Use 2 vs 3 Models

In a FastAPI DDD project, data can have up to three representations:

| Model                | File         | Purpose                                                |
| -------------------- | ------------ | ------------------------------------------------------ |
| **Pydantic Schema**  | `schemas.py` | API contract (input validation + output serialization) |
| **SQLAlchemy Model** | `models.py`  | Database persistence + entity identity                 |
| **Domain Entity**    | `domain.py`  | Pure business logic with invariants (dataclass)        |

### Default: 2 Models (90% of Projects)

For most FastAPI projects, the SQLAlchemy model serves double duty as entity + persistence:

```python
# models.py — Entity + Persistence (unified)
class Document(Base):
    __tablename__ = "documents"
    id: Mapped[UUID] = mapped_column(primary_key=True, default=uuid4)
    user_id: Mapped[UUID] = mapped_column(index=True)
    filename: Mapped[str]
    ocr_status: Mapped[str] = mapped_column(default="pending")

# schemas.py — API contracts (separate per direction)
class DocumentCreateRequest(BaseModel):
    description: str | None = None

class DocumentResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: UUID
    filename: str
    ocr_status: str
```

### Advanced: 3 Models (When Justified)

Add a pure domain entity **only when**:

- Business invariants are complex and must live in the entity (state machines, cross-field rules)
- The same domain logic is consumed by API services, Celery workers, and CLI tools
- You find the same `if order.status == "shipped": raise ...` check duplicated in 3+ files

```python
# domain.py — Pure entity (no ORM imports)
@dataclass
class Order:
    status: str
    shipped_at: datetime | None = None

    def cancel(self) -> None:
        """Cancel the order. Raises if not cancellable."""
        if self.status == "shipped":
            raise OrderAlreadyShippedError(self.id)
        self.status = "canceled"
        self.canceled_at = datetime.utcnow()

# mapping.py — Imperative mapping (SQLAlchemy 2.x)
registry.map_imperatively(Order, orders_table)
```

**Decision**: If your entities are mostly data containers with logic in services (which is the case for 90% of FastAPI projects), stick with 2 models. Add domain entities when the pain of duplicating business rules across service + worker + CLI becomes real.
