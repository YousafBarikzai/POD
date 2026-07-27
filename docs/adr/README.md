# Architecture Decision Records

Each ADR records one decision, the context that forced it, the options considered, and the consequences we accepted. Format: a compressed MADR.

**Status values:** `proposed` · `accepted` · `superseded by ADR-XXXX` · `deprecated`

| ADR | Title | Status | Affects |
|---|---|---|---|
| [0001](ADR-0001-multi-tenancy-from-day-one.md) | Multi-tenancy from day one | accepted | Schema, API, security |
| [0002](ADR-0002-postgres-as-primary-store.md) | PostgreSQL as the single primary store | accepted | Data, search, vectors |
| [0003](ADR-0003-durable-run-orchestration.md) | Durable, resumable run orchestration | accepted | Worker, reliability |
| [0004](ADR-0004-provider-adapter-boundary.md) | Provider adapter boundary | accepted | All integrations |
| [0005](ADR-0005-deterministic-scoring-llm-judgement.md) | Deterministic maths, LLM judgement | accepted | Scoring, AI |
| [0006](ADR-0006-versioned-scoring-config.md) | Versioned scoring configuration | accepted | Scoring, learning |
| [0007](ADR-0007-structured-ai-output.md) | Structured AI output only | accepted | AI |
| [0008](ADR-0008-human-gates.md) | Three mandatory human gates | accepted | Product, safety |
| [0009](ADR-0009-market-data-acquisition-posture.md) | Compliance-ranked data acquisition | accepted | Market data, legal |
| [0010](ADR-0010-ai-cost-budgeting.md) | Cost as a first-class budget | accepted | AI, economics |
| [0011](ADR-0011-trpc-internal-rest-public.md) | tRPC internally, REST publicly | accepted | API |
| [0012](ADR-0012-modular-monolith.md) | Modular monolith, not microservices | accepted | Services |
| [0013](ADR-0013-immutable-snapshots.md) | Immutable snapshots over mutable rows | accepted | Data model |
| [0014](ADR-0014-prisma-orm.md) | Prisma as ORM with a repository layer | accepted | Data access |

## When a new ADR is required

- The decision affects more than one package.
- The decision affects data durability, an external contract, or cost.
- The decision reverses or narrows a previous ADR.
- The decision adds a new external dependency.

An ADR is *not* required for choices inside a single package that can be reversed in an afternoon.
