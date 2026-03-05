# Service Layer Pattern

## The Three Layers

Every request flows through three layers with distinct responsibilities:

```
Router (HTTP)  →  Service (Business Logic)  →  Database (Persistence)
   thin               thick                      thin
```

### Router — Parse, Delegate, Respond

The router handles HTTP concerns **only**:

- Parse path/query/body parameters
- Call the service
- Return the response with correct status code
- Handle HTTP-specific errors (404, 409, etc.)

```python
# app/modules/documents/router.py
from typing import Annotated
from fastapi import APIRouter, Depends, UploadFile, status
from sqlalchemy.ext.asyncio import AsyncSession

from app.shared.types import DB, CurrentUser
from app.modules.documents.schemas import DocumentResponse, DocumentListResponse
from app.modules.documents.service import DocumentService

router = APIRouter(prefix="/documents", tags=["documents"])


@router.post("/", response_model=DocumentResponse, status_code=status.HTTP_201_CREATED)
async def upload_document(
    file: UploadFile,
    db: DB,
    current_user: CurrentUser,
) -> DocumentResponse:
    service = DocumentService(db)
    return await service.upload(file, user_id=current_user.id)


@router.get("/", response_model=DocumentListResponse)
async def list_documents(
    db: DB,
    current_user: CurrentUser,
    status_filter: str | None = None,
) -> DocumentListResponse:
    service = DocumentService(db)
    return await service.list_files(user_id=current_user.id, status=status_filter)
```

**Red flags in a router**: `db.execute()`, `select()`, `await db.commit()`, business conditionals, `if user.plan == ...`. These belong in the service.

### Service — Business Logic

The service contains the **domain logic** — what the application actually does:

```python
# app/modules/documents/service.py
from uuid import UUID
from fastapi import UploadFile
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.shared.exceptions import NotFoundError
from app.core.storage import StorageService
from app.modules.documents.models import Document
from app.modules.documents.schemas import DocumentResponse, DocumentListResponse
from app.modules.documents.validators import FileValidator


class DocumentService:
    def __init__(self, db: AsyncSession):
        self.db = db
        self.storage = StorageService()

    async def upload(self, file: UploadFile, *, user_id: UUID) -> DocumentResponse:
        # 1. Validate
        FileValidator.validate_file(file)

        # 2. Store file
        file_path = await self.storage.upload_file(file, user_id=str(user_id))

        # 3. Create DB record
        document = Document(
            user_id=user_id,
            filename=file.filename,
            file_path=file_path,
            content_type=file.content_type,
            file_size=file.size,
            ocr_status="pending",
        )
        self.db.add(document)
        await self.db.flush()
        await self.db.refresh(document)

        # 4. Dispatch background processing
        process_document.delay(str(document.id))

        return DocumentResponse.model_validate(document)

    async def list_files(
        self, *, user_id: UUID, status: str | None = None
    ) -> DocumentListResponse:
        query = select(Document).where(
            Document.user_id == user_id,
            Document.is_deleted == False,
        )
        if status:
            query = query.where(Document.ocr_status == status)

        result = await self.db.execute(query.order_by(Document.created_at.desc()))
        documents = result.scalars().all()

        return DocumentListResponse(
            items=[DocumentResponse.model_validate(doc) for doc in documents],
            total=len(documents),
        )

    async def get_by_id(self, document_id: UUID, *, user_id: UUID) -> DocumentResponse:
        document = await self._get_owned_document(document_id, user_id)
        return DocumentResponse.model_validate(document)

    async def soft_delete(self, document_id: UUID, *, user_id: UUID) -> None:
        document = await self._get_owned_document(document_id, user_id)
        document.is_deleted = True
        await self.db.flush()

    # ---- Private helpers ----

    async def _get_owned_document(self, document_id: UUID, user_id: UUID) -> Document:
        result = await self.db.execute(
            select(Document).where(
                Document.id == document_id,
                Document.user_id == user_id,
                Document.is_deleted == False,
            )
        )
        document = result.scalar_one_or_none()
        if not document:
            raise NotFoundError("Document", str(document_id))
        return document
```

## Service Patterns

### Constructor Injection

Pass the database session (and optionally other services) via constructor:

```python
class OrderService:
    def __init__(self, db: AsyncSession):
        self.db = db

# In the router:
service = OrderService(db)
```

For services that need infrastructure clients (storage, cache, email):

```python
class DocumentService:
    def __init__(self, db: AsyncSession):
        self.db = db
        self.storage = StorageService()       # Infrastructure
        self.search = SearchService()         # Infrastructure

    # Business methods use self.storage, self.search, self.db
```

### Service as FastAPI Dependency

For complex wiring, use a factory dependency:

```python
async def get_document_service(db: DB) -> DocumentService:
    return DocumentService(db)

DocService = Annotated[DocumentService, Depends(get_document_service)]

@router.post("/", response_model=DocumentResponse)
async def upload(file: UploadFile, service: DocService, current_user: CurrentUser):
    return await service.upload(file, user_id=current_user.id)
```

### Standalone Functions vs Class

Use a **class** when:

- The service needs shared state (db session, storage client)
- Multiple methods share private helpers
- The module has 3+ operations

Use **standalone functions** when:

- The logic is simple (1-2 functions)
- No shared state needed
- Used as utility in other modules

```python
# Standalone — fine for simple operations
async def create_stripe_customer(user: User) -> str:
    customer = stripe.Customer.create(email=user.email)
    return customer.id

# Class — better when state and multiple operations
class BillingService:
    def __init__(self, db: AsyncSession):
        self.db = db

    async def create_checkout(self, user: User, plan: str) -> str: ...
    async def get_status(self, user: User) -> SubscriptionStatus: ...
    async def handle_webhook(self, event: dict) -> None: ...
```

## Responsibility Boundaries

| Concern                                   | Where it belongs                             |
| ----------------------------------------- | -------------------------------------------- |
| HTTP parsing (path, query, body, headers) | Router                                       |
| Status codes and HTTP errors              | Router                                       |
| Input validation (schema-level)           | Schemas (Pydantic)                           |
| Business rules and orchestration          | Service                                      |
| Domain validation ("can this user do X?") | Service or Guards                            |
| Database queries                          | Service (simple) or Queries (complex)        |
| Transaction management (commit/rollback)  | Database dependency (`get_db`)               |
| External API calls                        | Service (wrapping infra client from `core/`) |
| Background task dispatch                  | Service                                      |
| Event publishing                          | Service (via extracted event helpers)        |

## Domain Exceptions (Mandatory)

Services **must** raise domain exceptions, never `HTTPException`. This decouples business logic from the HTTP layer and makes services reusable from workers, CLI tools, and tests without HTTP dependencies:

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

```python
# In service — raise domain exceptions, NOT HTTPException
class DocumentService:
    async def _get_owned_document(self, document_id: UUID, user_id: UUID) -> Document:
        result = await self.db.execute(
            select(Document).where(
                Document.id == document_id,
                Document.user_id == user_id,
                Document.is_deleted == False,
            )
        )
        document = result.scalar_one_or_none()
        if not document:
            raise NotFoundError("Document", str(document_id))
        return document
```

```python
# In router — translate domain exceptions to HTTP
from app.shared.exceptions import NotFoundError, LimitExceededError

@router.get("/{doc_id}", response_model=DocumentDetailResponse)
async def get_document(doc_id: UUID, db: DB, current_user: CurrentUser):
    service = DocumentService(db)
    try:
        return await service.get_by_id(doc_id, user_id=current_user.id)
    except NotFoundError:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "Document not found")
```

**Recommended**: Register global exception handlers in `main.py` to eliminate repetitive try/except in routers:

```python
# app/main.py
from app.shared.exceptions import (
    DomainError, NotFoundError, PermissionDeniedError,
    LimitExceededError, ConflictError, ServiceUnavailableError,
)

@app.exception_handler(NotFoundError)
async def not_found_handler(request, exc: NotFoundError):
    return JSONResponse(status_code=404, content={"code": exc.code, "message": exc.message})

@app.exception_handler(PermissionDeniedError)
async def permission_handler(request, exc: PermissionDeniedError):
    return JSONResponse(status_code=403, content={"code": exc.code, "message": exc.message})

@app.exception_handler(LimitExceededError)
async def limit_handler(request, exc: LimitExceededError):
    return JSONResponse(status_code=402, content={
        "code": exc.code, "message": exc.message,
        "limit": exc.limit, "current": exc.current,
    })

@app.exception_handler(ConflictError)
async def conflict_handler(request, exc: ConflictError):
    return JSONResponse(status_code=409, content={"code": exc.code, "message": exc.message})

@app.exception_handler(ServiceUnavailableError)
async def service_unavailable_handler(request, exc: ServiceUnavailableError):
    return JSONResponse(status_code=503, content={"code": exc.code, "message": exc.message})
```

**Rule of thumb**: If the exception describes a **business rule violation**, use a domain exception. If it describes an **HTTP concern** (malformed request, auth header missing), `HTTPException` is fine directly in the router or guard.

## What NOT to Put in Services

- HTTP concerns (`Request`, `Response`, headers, cookies)
- Framework-specific imports (`APIRouter`, `status` codes for response)
- `HTTPException` — **always** use domain exceptions instead (see Domain Exceptions above). Services must never import from `fastapi` except `UploadFile` when handling file uploads
- Direct infrastructure calls that should be in `core/` (raw Redis commands, raw S3 calls)
- Logic from another domain (don't import another module's service)

## Splitting a God Service

Services grow organically. Watch for these signals that a service needs splitting:

**Signals**:

- Service has **> 300 lines** or **> 10 methods**
- Methods span **clearly different business capabilities** (CRUD + processing + sharing + importing)
- The service constructor needs **> 3 infrastructure dependencies**

**How to split**: By **business capability**, not by arbitrary size:

```
app/modules/documents/
├── service.py            # Core CRUD: upload, get, list, delete
├── processing_service.py # OCR, reprocessing, PDF export
├── sharing_service.py    # Permissions, team sharing
└── import_service.py     # Bulk import, external sync
```

Each sub-service receives `AsyncSession` independently. The router (or `orchestration.py`) orchestrates them:

```python
@router.post("/batch-import", response_model=ImportResultResponse)
async def batch_import(data: ImportRequest, db: DB, current_user: CurrentUser):
    import_service = ImportService(db)
    return await import_service.bulk_import(data, user_id=current_user.id)
```

**Don't split** when: a service has many methods but they all operate on the same concept (e.g., 8 methods for document CRUD + filtering + sorting is fine — it's one capability).

## Performance Considerations

Abstraction layers (router → service → queries) add negligible overhead (~0.01ms per call). The real performance bottlenecks are N+1 queries, missing indexes, and undersized connection pools. See `references/performance.md` for detailed guidance on:

- Real cost of each layer measured in latency
- Pydantic V2 optimization tips (`model_validate_json()`)
- Connection pool sizing for production
- When abstraction actually hurts (and what to do about it)
