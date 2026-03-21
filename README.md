# FastAPI DDD

Pragmatic Domain-Driven Design skill for AI coding agents. Guides modular FastAPI backend architecture with async SQLAlchemy, Pydantic V2, and clean separation of concerns — without over-engineering.

Compatible with any tool that supports [Skills](https://skills.sh) — Claude Code, Cursor, Copilot, Roo Code, Cline, and others.

## Install

```bash
npx skills add zeine-inc/fastapi-ddd
```

## What it does

When triggered, the agent acts as a senior backend architect that applies DDD pragmatically. It helps with:

- Structuring new FastAPI backends from scratch
- Adding domain modules to existing projects
- Refactoring monolithic routers into bounded contexts
- Reviewing architecture for DDD alignment
- Deciding where code belongs (`core/` vs `shared/` vs `modules/`)

## Canonical layout

```
app/
├── main.py          # Composition root — registers routers
├── orm.py           # ORM registry — registers all module models
├── core/            # Infrastructure (database, auth, config)
├── shared/          # Cross-module contracts (schemas, exceptions, types, mixins)
├── modules/         # Domain modules (bounded contexts)
│   ├── auth/
│   ├── documents/
│   └── billing/
└── workers/         # Background task processors
```

## Reference guides

| Topic | When to load |
| --- | --- |
| Module Structure | Creating modules, project layout, ORM registry |
| Service Layer | Writing services, splitting router logic, DI |
| Schemas & DTOs | Pydantic models, request/response types |
| Guards & Dependencies | Auth, plan enforcement, permissions |
| Queries & Data Access | Database queries, multi-tenancy, eager loading |
| Events & Integration | Pub/sub, external services, workers |
| Anti-Patterns | Code review, refactoring decisions |
| Testing | Service tests, fixtures, integration patterns |
| Caching | Redis, HTTP cache headers, invalidation |
| Resilience | Retry, circuit breaker, timeouts |
| Performance | Pool sizing, lazy vs eager loading, Pydantic V2 |

## License

MIT
