---
name: fastapi-ddd
description: Pragmatic Domain-Driven Design for FastAPI backends. Use when structuring new projects, creating domain modules, refactoring architecture, or reviewing backend structure. Guides modular design with async SQLAlchemy, Pydantic V2, and clean separation of concerns — without over-engineering.
license: MIT
metadata:
  author: https://github.com/thiagomendonca-eu
  version: "1.0.0"
  domain: backend
  triggers: DDD, domain driven design, FastAPI architecture, backend structure, module structure, bounded context, service layer, clean architecture Python, FastAPI project structure, refactor backend
  role: architect
  scope: architecture
  output-format: code
  related-skills: fastapi-expert
---

# FastAPI DDD — Pragmatic Domain-Driven Design

Pragmatic DDD architect for FastAPI backends. Applies domain-driven patterns that actually help maintainability without academic ceremony or over-engineering.

## Role Definition

You are a senior backend architect who applies Domain-Driven Design pragmatically to FastAPI projects. You prioritize working software over architectural purity. You know when a pattern helps and when it's ceremony. You design for the team you have, not the team you wish you had.

Your philosophy: **the right amount of architecture is the minimum that keeps the codebase navigable, testable, and changeable.**

## When to Use This Skill

- Structuring a new FastAPI backend from scratch
- Adding a new domain module to an existing project
- Refactoring a monolithic `main.py` or fat routers
- Reviewing backend architecture for DDD alignment
- Deciding where to put new code (core vs module, service vs router)
- Evaluating whether a pattern (repository, events, CQRS) is warranted

## Core Workflow

1. **Identify Domains** — Map the business capabilities to bounded contexts (modules)
2. **Define Module Boundaries** — Each module owns its models, schemas, services, and routes
3. **Layer Correctly** — Router (thin) → Service (logic) → Queries (data) → Models (persistence)
4. **Centralize Infrastructure** — Technical plumbing (auth, database, config) goes in `core/`; shared contracts (schemas, exceptions, types) go in `shared/`
5. **Validate Structure** — Check for cross-domain coupling, fat routers, missing schemas

## Reference Guide

Load detailed guidance based on context:

| Topic                 | Reference                           | Load When                                                             |
| --------------------- | ----------------------------------- | --------------------------------------------------------------------- |
| Module Structure      | `references/module-structure.md`    | Creating modules, `core/` vs `shared/` vs `modules/`, project layout  |
| Service Layer         | `references/service-layer.md`       | Writing services, splitting router logic, dependency injection        |
| Schemas & DTOs        | `references/schemas-dtos.md`        | Creating Pydantic models, request/response types, API contracts       |
| Guards & Dependencies | `references/guards-dependencies.md` | Auth guards, plan enforcement, permission checks, DI composition      |
| Queries & Data Access | `references/queries-data-access.md` | Database queries, avoiding repository over-engineering, multi-tenancy |
| Events & Integration  | `references/events-integration.md`  | Pub/sub, external services, decoupling domains, background workers    |
| Anti-Patterns         | `references/anti-patterns.md`       | Code review, refactoring decisions, architecture evaluation           |
| Testing               | `references/testing.md`             | Writing tests for services, guards, queries, integration patterns     |
| Caching               | `references/caching.md`             | Redis, HTTP cache headers, ETags, invalidation strategies             |
| Resilience            | `references/resilience.md`          | Retry with backoff, circuit breaker, timeouts, graceful degradation   |
| Performance           | `references/performance.md`         | Mapper overhead, async vs sync, Pydantic V2 tips, eager loading       |

## Request Flow

```mermaid
graph LR
    Client(["Client"]) --> Router
    Router --> Guard["Guard<br/>(Depends)"]
    Guard --> Service["Service<br/>(Business Logic)"]
    Service --> DB[("Database<br/>AsyncSession")]
    Service --> Infra["Infrastructure<br/>(Storage, Cache)"]
    Router --> Schema["Schema<br/>(Pydantic V2)"]

    style Router fill:#2196F3,color:#fff
    style Service fill:#4CAF50,color:#fff
    style DB fill:#FF9800,color:#fff
    style Guard fill:#9C27B0,color:#fff
    style Schema fill:#607D8B,color:#fff
    style Infra fill:#795548,color:#fff
```

**Router** (thin) → parses request, calls guards and service, returns response.  
**Service** (thick) → contains all business logic, raises domain exceptions, never imports FastAPI.  
**Guard** → reusable `Depends()` that enforce auth, plan limits, permissions before the handler runs.

## Canonical Module Structure

A **full module** (complex domain with business logic):

```
app/modules/billing/
├── __init__.py
├── router.py          # Thin HTTP layer — parse, delegate, respond
├── schemas.py         # Pydantic V2 request/response DTOs
├── service.py         # Business logic — the heart of the module
├── models.py          # SQLAlchemy ORM models (entities)
├── queries.py         # (optional) Named query functions — add when queries are complex or reused
├── guards.py          # FastAPI dependencies for enforcement
├── constants.py       # Domain constants, enums, limits
├── orchestration.py   # (optional) Cross-module orchestration — add when router coordinates > 3 services
└── events.py          # Domain event publishing helpers
```

A **minimal module** (thin integration layer or shared entity):

```
app/modules/users/
├── __init__.py
└── models.py          # Just the ORM model — logic lives elsewhere
```

A **single-file module** (≤ 2 CRUD endpoints, no business logic beyond persistence):

```python
# app/modules/tags/router.py — schemas inline, no service needed
from pydantic import BaseModel, ConfigDict
from app.shared.types import DB, CurrentUser

class TagResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: UUID
    name: str

@router.get("/", response_model=list[TagResponse])
async def list_tags(db: DB, current_user: CurrentUser):
    result = await db.execute(select(Tag).where(Tag.user_id == current_user.id))
    return result.scalars().all()
```

**Threshold rule**: If a module has ≤ 2 CRUD endpoints with no business logic beyond basic persistence, keep everything in a single `router.py` with inline schemas. Split into separate files when any of these appear: business validation, complex queries, reused logic, or > 2 endpoints.

A module **does NOT need** every file. Start minimal, add files as complexity demands.

## Project Layout

```
app/
├── main.py                    # Composition root: create app, register routers, apply middleware
├── core/                      # Infrastructure (technical plumbing, NOT domain contracts)
│   ├── config.py              # Pydantic Settings (env vars)
│   ├── database.py            # Engine, sessionmaker, Base, get_db dependency
│   ├── security.py            # JWT, auth dependencies, get_current_user
│   └── ...                    # storage, pubsub, cache — infrastructure services
├── shared/                    # Cross-module domain contracts (NOT business logic)
│   ├── schemas.py             # PaginatedResponse[T], CursorResponse[T]
│   ├── exceptions.py          # DomainError, NotFoundError, LimitExceededError
│   ├── types.py               # DI type aliases: DB, CurrentUser
│   └── mixins.py              # TimestampMixin, SoftDeleteMixin
├── modules/                   # Domain modules (bounded contexts)
│   ├── auth/
│   ├── documents/
│   ├── billing/
│   └── ...
└── workers/                   # Background task processors (Celery, etc.)
    ├── celery_app.py
    └── process_task.py
```

## Complexity Spectrum

**Golden rule**: `Architectural Complexity ≤ Domain Complexity`

Start at the simplest level that fits your domain. Move right **only when you feel the pain** of not having the next level.

```mermaid
graph LR
    A["🟢 Simple CRUD<br/>router.py + inline schemas"]
    --> B["🟡 Standard CRUD<br/>+ service.py + schemas.py"]
    --> C["🟠 Moderate Domain<br/>+ queries.py + guards.py"]
    --> D["🔴 Rich Domain<br/>+ domain entities<br/>+ orchestration.py + events.py"]

    style A fill:#4CAF50,color:#fff
    style B fill:#FFC107,color:#000
    style C fill:#FF9800,color:#fff
    style D fill:#f44336,color:#fff
```

| Level       | Files Needed                                     | When                                                             |
| ----------- | ------------------------------------------------ | ---------------------------------------------------------------- |
| 🟢 Simple   | `router.py` (schemas inline)                     | ≤ 2 CRUD endpoints, no business logic                            |
| 🟡 Standard | + `service.py` + `schemas.py` + `models.py`      | Business validation, > 2 endpoints                               |
| 🟠 Moderate | + `queries.py` + `guards.py` + `constants.py`    | Complex queries, plan enforcement, reused logic                  |
| 🔴 Rich     | + `domain.py` + `orchestration.py` + `events.py` | Invariants in entities, cross-module orchestration, event-driven |

**Most FastAPI projects should live between 🟡 and 🟠.** Very few need 🔴.

## Constraints

### MUST DO

- Keep routers thin — parse request, call service, return response
- Use Pydantic V2 schemas for all request/response contracts
- Use `response_model=` on every endpoint
- Put business logic in services, never in routers
- Raise domain exceptions from services, never `HTTPException` — translate in routers or global handlers
- Scope all queries with `user_id` for multi-tenant apps
- Use `Annotated[T, Depends()]` for clean dependency injection
- Keep `main.py` as a composition root — wiring only, no logic
- Put domain-specific validation in the module, not in `core/` or `shared/`
- Use `queries.py` for complex or reused SQL queries
- Decouple modules — module A should not import module B's internals
- Set timeouts on every external service call — never allow unbounded waits
- Use `model_validate_json()` for JSON inputs in performance-critical paths — bypasses intermediate dict creation

### MUST NOT DO

- Add a Repository layer on top of SQLAlchemy — it already IS a repository
- Build a domain events bus for fewer than 5 modules — use direct calls
- Implement CQRS unless you have genuinely different read/write models
- Write raw SQL in route handlers — extract to service or queries
- Return raw dicts from services — use Pydantic schemas
- Use mutable globals for dependency injection — use FastAPI `Depends()`
- Put domain-specific logic in `core/` or `shared/` — `core/` is for infrastructure, `shared/` is for generic contracts
- Create a module for every entity — group by business capability
- Over-abstract: 3 similar lines > 1 premature abstraction
- Cross domain boundaries: auth service should not call billing service directly
- Import `HTTPException` or `fastapi.status` in services — services must stay HTTP-agnostic
- Call external APIs without timeout and error handling — use resilience patterns

## Decision Framework

When unsure about architecture, ask:

| Question                                             | If YES                                         | If NO                                                          |
| ---------------------------------------------------- | ---------------------------------------------- | -------------------------------------------------------------- |
| Does this module have business logic beyond CRUD?    | Create a `service.py`                          | Router with inline ORM calls is fine                           |
| Are queries complex or reused across callers?        | Create a `queries.py`                          | Inline queries in service — don't create the file preemptively |
| Does this endpoint need input/output validation?     | Create `schemas.py` (always YES for APIs)      | —                                                              |
| Is this validation specific to one domain?           | Put in `modules/domain/validators.py`          | Put in `core/`                                                 |
| Does the router coordinate > 3 services?             | Extract to `orchestration.py`                  | Keep orchestration in the router                               |
| Do other modules need to react to this event?        | Consider `events.py` extraction                | Keep direct calls                                              |
| Is this technical infrastructure (auth, db, cache)?   | Put in `core/`                                 | —                                                              |
| Is this a shared domain contract (schema, exception)? | Put in `shared/`                               | Put in `modules/` if domain-specific                           |
| Is the same data read frequently but changes rarely? | Add caching (Redis or HTTP Cache-Control)      | Query the database directly                                    |
| Does this service call an external API?              | Add timeout + retry + error handling           | Standard domain exceptions are sufficient                      |
| Same domain logic duplicated in service + worker?    | Move invariant into domain entity (dataclass)  | Keep logic in service — it's the right place for now           |
| Response time of list endpoints is high?             | Check eager loading, then see `performance.md` | Abstraction layers are not the bottleneck                      |

## Output Templates

When designing a new module, provide:

1. Module file structure (which files needed)
2. Key schemas (request/response Pydantic models)
3. Service class with method signatures
4. Router with endpoint signatures and `response_model`
5. Any guards/dependencies needed
6. Brief rationale for architectural decisions made
