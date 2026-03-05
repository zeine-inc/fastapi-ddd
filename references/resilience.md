# Resilience Patterns

External services fail. Networks timeout. APIs rate-limit. Resilience patterns keep your application functional when dependencies degrade.

## When to Add Resilience

Add resilience patterns when your service calls:

- External APIs (Stripe, OpenAI, SendGrid, S3)
- Other microservices over HTTP/gRPC
- Infrastructure that can become slow (search indexes, vector databases)

Don't add resilience for:

- Local database calls (handled by connection pool + transaction rollback)
- In-process function calls
- Background task dispatching (task queues have their own retry semantics)

## Timeout — The First Line of Defense

Every external call needs a timeout. Without one, a stalled dependency blocks your worker forever:

```python
# app/core/http_client.py
import httpx

# Shared client with sensible defaults — reuse across services
http_client = httpx.AsyncClient(
    timeout=httpx.Timeout(
        connect=5.0,    # Time to establish connection
        read=30.0,      # Time to receive response
        write=10.0,     # Time to send request body
        pool=5.0,       # Time to acquire connection from pool
    ),
    limits=httpx.Limits(
        max_connections=100,
        max_keepalive_connections=20,
    ),
)
```

```python
# In service — always use the shared client
from app.core.http_client import http_client

class OCRService:
    async def process(self, file_url: str) -> str:
        response = await http_client.post(
            "https://api.ocr-provider.com/v1/process",
            json={"url": file_url},
            timeout=60.0,  # Override for slow endpoints
        )
        response.raise_for_status()
        return response.json()["text"]
```

### Timeout Guidelines

| External Call               | Timeout                  | Why                                     |
| --------------------------- | ------------------------ | --------------------------------------- |
| Payment APIs (Stripe)       | 10-15s                   | Payments are critical, allow extra time |
| LLM APIs (OpenAI)           | 60-120s                  | Generative models are slow              |
| Email/notification services | 5-10s                    | Fire-and-forget, don't block            |
| Storage operations (S3)     | 30s upload, 10s download | Depends on file size                    |
| Search/vector queries       | 5-10s                    | Should be fast, fail early if not       |

## Retry with Exponential Backoff

Retries recover from transient failures (network blips, 503s, rate limits). Always use backoff to avoid thundering herd:

```python
# app/core/resilience.py
import asyncio
import logging
from typing import TypeVar, Callable, Awaitable
from functools import wraps

logger = logging.getLogger(__name__)

T = TypeVar("T")


async def retry_async(
    func: Callable[..., Awaitable[T]],
    *args,
    max_retries: int = 3,
    base_delay: float = 1.0,
    max_delay: float = 30.0,
    retryable_exceptions: tuple[type[Exception], ...] = (Exception,),
    **kwargs,
) -> T:
    """Retry an async function with exponential backoff.

    Args:
        func: Async function to retry.
        max_retries: Maximum number of retry attempts (not counting the initial call).
        base_delay: Initial delay in seconds (doubles each retry).
        max_delay: Maximum delay cap in seconds.
        retryable_exceptions: Exception types that trigger a retry.
    """
    last_exception: Exception | None = None

    for attempt in range(max_retries + 1):
        try:
            return await func(*args, **kwargs)
        except retryable_exceptions as exc:
            last_exception = exc
            if attempt == max_retries:
                break

            delay = min(base_delay * (2 ** attempt), max_delay)
            logger.warning(
                "Retry %d/%d for %s after %.1fs: %s",
                attempt + 1, max_retries, func.__name__, delay, exc,
            )
            await asyncio.sleep(delay)

    raise last_exception  # type: ignore[misc]
```

### Usage in Services

```python
from app.core.resilience import retry_async

class BillingService:
    def __init__(self, db: AsyncSession):
        self.db = db

    async def create_customer(self, user: User) -> str:
        """Create Stripe customer with retry for transient failures."""
        return await retry_async(
            self._create_stripe_customer,
            user,
            max_retries=3,
            base_delay=1.0,
            retryable_exceptions=(httpx.TransportError, httpx.HTTPStatusError),
        )

    async def _create_stripe_customer(self, user: User) -> str:
        response = await http_client.post(
            "https://api.stripe.com/v1/customers",
            data={"email": user.email},
            headers={"Authorization": f"Bearer {settings.stripe_key}"},
        )
        response.raise_for_status()
        return response.json()["id"]
```

### Which Errors to Retry

| Retry                                          | Don't Retry                                     |
| ---------------------------------------------- | ----------------------------------------------- |
| `httpx.TransportError` (connection reset, DNS) | `httpx.HTTPStatusError` with 400, 401, 403, 404 |
| HTTP 429 (rate limited)                        | HTTP 422 (validation error)                     |
| HTTP 500, 502, 503, 504                        | Business logic errors                           |
| Timeout errors                                 | Authentication failures                         |

```python
import httpx

RETRYABLE_STATUS = {429, 500, 502, 503, 504}


def is_retryable(exc: Exception) -> bool:
    """Check if an exception is worth retrying."""
    if isinstance(exc, httpx.TransportError):
        return True
    if isinstance(exc, httpx.HTTPStatusError):
        return exc.response.status_code in RETRYABLE_STATUS
    return False
```

## Circuit Breaker

When a dependency is persistently failing, stop sending requests to let it recover. Prevents cascading failures and wasted timeouts:

```python
# app/core/resilience.py
import time
from enum import Enum


class CircuitState(str, Enum):
    CLOSED = "closed"       # Normal operation
    OPEN = "open"           # Failing — reject calls immediately
    HALF_OPEN = "half_open" # Testing if service recovered


class CircuitBreaker:
    """Simple circuit breaker for external service calls.

    Usage:
        ocr_circuit = CircuitBreaker(failure_threshold=5, recovery_timeout=60)

        async def call_ocr(file_url: str) -> str:
            if not ocr_circuit.allow_request():
                raise ServiceUnavailableError("OCR service temporarily unavailable")
            try:
                result = await ocr_client.process(file_url)
                ocr_circuit.record_success()
                return result
            except Exception as exc:
                ocr_circuit.record_failure()
                raise
    """

    def __init__(self, failure_threshold: int = 5, recovery_timeout: float = 60.0):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.last_failure_time: float = 0.0

    def allow_request(self) -> bool:
        if self.state == CircuitState.CLOSED:
            return True
        if self.state == CircuitState.OPEN:
            if time.monotonic() - self.last_failure_time >= self.recovery_timeout:
                self.state = CircuitState.HALF_OPEN
                return True
            return False
        # HALF_OPEN: allow one request to test
        return True

    def record_success(self) -> None:
        self.failure_count = 0
        self.state = CircuitState.CLOSED

    def record_failure(self) -> None:
        self.failure_count += 1
        self.last_failure_time = time.monotonic()
        if self.failure_count >= self.failure_threshold:
            self.state = CircuitState.OPEN
```

### Where to Place Circuit Breakers

```python
# app/modules/documents/service.py — per external dependency
from app.core.resilience import CircuitBreaker
from app.shared.exceptions import ServiceUnavailableError

_ocr_circuit = CircuitBreaker(failure_threshold=5, recovery_timeout=60)


class DocumentService:
    async def process_ocr(self, document_id: UUID) -> str:
        if not _ocr_circuit.allow_request():
            raise ServiceUnavailableError("OCR service temporarily unavailable")

        try:
            result = await self._call_ocr_api(document_id)
            _ocr_circuit.record_success()
            return result
        except (httpx.TransportError, httpx.HTTPStatusError) as exc:
            _ocr_circuit.record_failure()
            raise
```

**When to use circuit breakers**: Only for dependencies that can fail persistently (external APIs, third-party services). Not needed for your own database or Redis.

## Resilient Cross-Module Orchestration

Replace fragile `try/except Exception: logger.warning(...)` with explicit resilience:

```python
# ❌ Fragile — swallows all errors, no retry, no visibility
@router.post("/register", response_model=AuthResponse)
async def register(data: RegisterRequest, db: DB):
    auth_service = AuthService(db)
    user, is_new = await auth_service.create_user(data)

    if is_new:
        try:
            await create_stripe_customer(user)
        except Exception:
            logger.warning("Stripe failed")  # Lost in logs, never retried

    return AuthResponse(...)


# ✅ Resilient — retryable operations with clear degradation
@router.post("/register", response_model=AuthResponse)
async def register(
    data: RegisterRequest,
    db: DB,
    background_tasks: BackgroundTasks,
):
    auth_service = AuthService(db)
    user, is_new = await auth_service.create_user(data)

    if is_new:
        # Critical: retry in background if it fails
        background_tasks.add_task(
            create_stripe_customer_with_retry, user.id
        )

    return AuthResponse(...)


# In billing service:
async def create_stripe_customer_with_retry(user_id: UUID) -> None:
    """Background task that retries Stripe customer creation."""
    async with get_async_session() as db:
        user = await db.get(User, user_id)
        if not user:
            return

        await retry_async(
            _create_stripe_customer,
            user,
            max_retries=5,
            base_delay=2.0,
            retryable_exceptions=(httpx.TransportError, httpx.HTTPStatusError),
        )
```

## Worker Retry with Backoff

Celery tasks should use exponential backoff, not fixed retries:

```python
# app/workers/ocr_task.py

@celery_app.task(
    bind=True,
    max_retries=5,
    default_retry_delay=None,  # We set it manually with backoff
)
def process_document(self, document_id: str):
    try:
        # ... processing logic ...
        pass
    except TransientError as exc:
        # Exponential backoff: 2s, 4s, 8s, 16s, 32s
        delay = min(2 ** self.request.retries, 300)
        raise self.retry(exc=exc, countdown=delay)
    except PermanentError:
        # Don't retry — mark as failed
        with get_sync_session() as session:
            doc = session.get(Document, document_id)
            doc.ocr_status = "failed"
            session.commit()
```

## Graceful Degradation Pattern

When a non-critical dependency fails, return partial results instead of a 500:

```python
class DashboardService:
    async def get_dashboard(self, *, user_id: UUID) -> DashboardResponse:
        # Core data — must succeed
        documents = await self._get_recent_documents(user_id)

        # Non-critical — degrade gracefully
        try:
            usage_stats = await self._get_usage_stats(user_id)
        except ServiceUnavailableError:
            usage_stats = None  # Frontend shows "Stats unavailable"

        return DashboardResponse(
            documents=documents,
            usage_stats=usage_stats,  # Optional field in schema
        )
```

## Decision Table

| Scenario                | Pattern                            | Config                                |
| ----------------------- | ---------------------------------- | ------------------------------------- |
| External API call       | Timeout + Retry                    | 3 retries, 1s base delay              |
| Payment processing      | Timeout + Retry (careful)          | 2 retries, 2s base delay, only on 5xx |
| LLM / slow API          | Long timeout + Circuit breaker     | 120s timeout, 5 failures → open       |
| Email sending           | Fire-and-forget + Background retry | BackgroundTasks with retry_async      |
| Webhook delivery        | Retry with dead letter             | 5 retries, exponential, then DLQ      |
| Non-critical enrichment | Timeout + Graceful degradation     | 3s timeout, return partial on failure |
