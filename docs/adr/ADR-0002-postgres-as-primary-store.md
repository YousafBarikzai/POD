# ADR-0002 — PostgreSQL as the single primary store

**Status:** accepted · **Date:** Phase 0

## Context

The system needs: relational integrity across ~40 entities with deep lineage; time-series snapshots at high volume; full-text search over titles, tags and descriptions; vector similarity for concept and image embeddings; JSON storage for provider payloads and feature vectors; and analytical aggregation for reports.

The reflexive architecture would be Postgres + Elasticsearch + Pinecone + a warehouse. That is four systems to operate, four consistency boundaries, and four failure modes — for a product with one user.

## Decision

PostgreSQL 16 is the single primary store through Stage 3, using:

| Need | Mechanism |
|---|---|
| Relational integrity | Native FKs, no soft referential integrity |
| Time series | Declarative range partitioning by month, from day one |
| Full-text search | `tsvector` generated columns + GIN, `pg_trgm` for fuzzy matching |
| Vector similarity | `pgvector` with HNSW indexes |
| Flexible payloads | `jsonb` with GIN where queried |
| Analytics | Read replicas + materialised views |

## Options considered

| Option | Verdict |
|---|---|
| **Postgres for everything** | **Accepted** |
| Postgres + Elasticsearch | Rejected — our search is scoped to one workspace's tens of thousands of rows, well within `tsvector` capability. ES adds a sync boundary and an operational burden for no current gain. |
| Postgres + dedicated vector DB | Rejected — pgvector with HNSW handles millions of vectors comfortably. A separate service adds a consistency problem (embeddings that must stay in sync with rows). |
| Postgres + warehouse from the start | Rejected — replicas answer analytical queries through Stage 3. |

## Consequences

**Positive**
- One backup, one restore, one failover, one connection story, one set of transactions. Snapshots, vectors and relational data are consistent by construction.
- Local development is a single container.
- Vector search joins directly to relational filters (`WHERE workspace_id = … ORDER BY embedding <=> …`), which a separate vector service makes awkward.

**Negative / accepted costs**
- pgvector index memory becomes a constraint at very high vector counts — mitigated by partitioning vectors by workspace and, if needed, quantisation.
- `tsvector` ranking is less sophisticated than a dedicated search engine. Acceptable: our search is navigational, not relevance-critical.
- Analytical queries compete with transactional load — mitigated by explicit read/write client separation from day one.

**Exit criteria (when to revisit):** vector index memory exceeds available RAM after partitioning; or analytical query load measurably degrades transactional p95 despite replicas. Both are Stage 3–4 concerns, and both have a documented migration path in doc 16 §13.
