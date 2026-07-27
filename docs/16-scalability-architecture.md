# 16 — Scalability Architecture

**Version:** 1.0

---

## 1. Scaling stages

The architecture is designed so that each stage is reached by **adding capacity or splitting an existing seam**, never by rewriting.

| Stage | Users | Runs/day | Listings tracked | Architecture |
|---|---|---|---|---|
| **1 — Personal** | 1 | 3 | 1,000 | Single web instance, single worker, single Postgres, single Redis |
| **2 — Early SaaS** | 50 | 150 | 50,000 | 2–3 web, 3–5 workers, Postgres + read replica, Redis, CDN |
| **3 — Growth** | 1,000 | 3,000 | 1,000,000 | Autoscaled web, queue-sharded workers, primary + 2 replicas, PgBouncer, partitioned hot tables, object-storage tiering |
| **4 — Scale** | 10,000 | 30,000 | 10,000,000 | Regional deployments, Citus or logical sharding by workspace, dedicated analytics store, Temporal for orchestration, separate vector service |

---

## 2. What actually breaks first

Ranked by when it bites, not by how interesting it is:

| # | Bottleneck | Bites at | Signal | Fix |
|---|---|---|---|---|
| 1 | **Third-party rate limits** (Etsy 10k/day, Printify, Ideogram, AI TPM) | Stage 2 | 429 rate rising, queue age growing | Per-tenant quotas, priority classes, aggressive caching, per-tenant provider credentials (each tenant brings their own Etsy app quota) |
| 2 | **AI cost per run** | Stage 2 | Gross margin per workspace falling | Tier discipline, prompt caching, aggregation, shared cross-tenant caches for niche-level analysis |
| 3 | **Worker CPU** during statistics and image processing | Stage 2–3 | Queue age, CPU saturation | Horizontal worker scaling, queue sharding, move image processing to a dedicated worker class |
| 4 | **Postgres write throughput** on `listing_snapshots` | Stage 3 | Write latency, bloat, autovacuum lag | Partitioning (already declared), batch inserts, `COPY` for bulk ingest, moving raw snapshots to columnar cold storage |
| 5 | **Postgres connection count** | Stage 3 | Connection saturation | PgBouncer in transaction mode; per-service pool caps |
| 6 | **Analytical query latency** on large tables | Stage 3 | Report page p95 | Read replicas, materialised aggregates, pre-computed report artefacts |
| 7 | **Vector index size and build time** | Stage 3–4 | HNSW memory, recall degradation | Partition vectors by workspace; move to a dedicated vector service if pgvector memory becomes the constraint |
| 8 | **Object storage volume** | Stage 3–4 | Cost | Lifecycle tiering, aggressive dedupe, thumbnail-only retention for competitor images |
| 9 | **Orchestrator coordination** | Stage 4 | Step scheduling latency, lock contention | Migrate the run state machine to Temporal (the DAG definition ports directly) |

**Note the ordering.** The first two bottlenecks are *external and economic*, not internal and technical. Most scaling documents get this backwards. For this product, the limiting factor is provider quota and AI cost long before it is CPU or database throughput — which is why doc 10's cost engineering and the per-tenant credential model in §7 below matter more than any sharding plan.

---

## 3. Stateless tiers and horizontal scaling

| Tier | Stateless? | Scaling trigger | Notes |
|---|---|---|---|
| Web (Next.js) | ✅ fully | CPU > 60% or p95 latency > budget | Sessions in Redis, no in-process cache of mutable state |
| Worker | ✅ fully | Queue age > 2 min or depth > 200 | Scale per queue class independently |
| Scheduler | ❌ singleton | — | Redis lock ensures one active leader; failover in < 30 s |
| SSE endpoints | ✅ | Connection count | Fan-out via Redis pub/sub, so any instance can serve any run's stream |

**Requirement (NFR-S3):** no component holds run state in process memory across steps. Verified by a chaos test that kills a worker mid-run and asserts identical final output.

---

## 4. Queue scaling

```
Stage 1:  one worker process, all queues,  concurrency 4/6/2/2/3/2
Stage 2:  worker classes — analysis · vision · artwork · publish · sync
Stage 3:  per-class autoscaling on queue age; per-tenant fair scheduling
Stage 4:  regional queue clusters; workspace-affinity routing for cache locality
```

**Fair scheduling (Stage 2 onwards):** a single tenant running 50 deep research runs must not starve everyone else. Implemented as weighted round-robin over per-tenant sub-queues with a per-tenant concurrency cap derived from their plan. This is the multi-tenant behaviour most likely to be discovered too late, so it is built at Stage 2, not Stage 3.

**Priority classes:** user-initiated interactive work (regenerate a concept, publish) preempts background work (catalogue sync, performance sync, recalibration).

---

## 5. Database scaling

### 5.1 Partitioning (built in from day one)

`listing_snapshots`, `performance_snapshots`, `provider_calls`, `ai_calls`, `run_events` are declared `PARTITION BY RANGE (created_at)` with monthly partitions, managed by a maintenance job that pre-creates the next two months and drops or archives beyond the retention window.

Cost of doing this now: one migration. Cost of doing it at 1B rows: an outage.

### 5.2 Read replicas (Stage 2)

| Workload | Target |
|---|---|
| Interactive reads for reports and dashboards | Replica |
| Analytics and exports | Replica |
| Run orchestration, all writes, read-after-write paths | Primary |

Routing is explicit at the repository layer (`db.read.*` vs `db.write.*`), never inferred, so replica lag can never silently break a read-after-write invariant.

### 5.3 Connection pooling (Stage 3)

PgBouncer in transaction mode, per-service pools sized to `(cores × 2) + effective_spindle_count`, with Prisma configured for the constraints of transaction pooling (no session-level state, no prepared-statement reliance).

### 5.4 Sharding (Stage 4)

Workspace id is the natural shard key: **there are no cross-workspace queries in the entire application** except platform-level admin analytics, which move to a separate analytics store. This is a direct consequence of the day-one tenancy decision, and it is what makes Stage 4 tractable.

Options, in order of preference: Citus distributed tables on `workspace_id` → logical sharding with a routing layer → regional isolation. Shared reference data (blueprints, print providers, taxonomy) is replicated to every shard.

### 5.5 Cold storage tiering (Stage 3)

Raw `listing_snapshots` older than 180 days move to Parquet in object storage, queryable on demand. Derived aggregates (`success_factors`, `keyword_stats`, `outcome_features`) stay hot indefinitely — they are small and are what the product actually reads.

---

## 6. Caching hierarchy

| Layer | Contents | Hit rate target | Invalidation |
|---|---|---|---|
| CDN | Static assets, signed images | > 95% | Content hash |
| Browser (React Query) | Report payloads, tables | > 70% | Key-based, per the cache map |
| Redis — session | Sessions, rate limits, locks | ~100% | TTL |
| Redis — domain | Blueprint catalogue, taxonomy, provider health | > 90% | TTL + sync job |
| Redis — expensive lookups | Trademark registry results | > 60% | 30-day TTL by normalised term |
| Postgres — vision cache | `style_profiles` by `(image_hash, extractor_version)` | > 40% steady state | Extractor version bump |
| Postgres — AI cache | Deterministic-purpose responses by `input_hash` | 15–30% | Prompt version bump |
| Object storage | Assets by content hash | ~100% | Never (immutable) |

**Cross-tenant cache opportunity (Stage 2+):** competitor style profiles and trademark lookups are *market facts*, not tenant data. Sharing them across tenants — with strict scoping of anything derived or tenant-specific — collapses marginal cost for popular niches. Two tenants researching "Gardening T-Shirts" in the same week should not pay twice to classify the same 400 thumbnails. This requires a careful boundary: shared cache holds only `(image_hash → style attributes)` and `(term → registry result)`, never scores, cohorts, factors or concepts. It is the single largest unit-economics improvement available at SaaS scale, and the schema already supports it because `style_profiles` is keyed by content hash.

---

## 7. External provider scaling

**The key structural insight:** in SaaS mode, each tenant connects **their own** Etsy app credentials, Printify token and (optionally) their own AI/Ideogram keys. Provider quota therefore scales with tenant count rather than being a shared ceiling.

| Provider | Phase 1 | SaaS |
|---|---|---|
| Etsy | Operator's app, 10k/day | Per-tenant OAuth app or per-tenant quota tracking; reserved publish allocation per tenant |
| Printify | Operator's token | Per-tenant token |
| Ideogram | Shared platform key | Shared key with per-tenant metering, or BYO-key on higher plans |
| AI | Shared platform key | Shared with per-tenant metering; enterprise tier may BYO |
| Market data | Operator's own CSV/subscription | Per-tenant; the CSV path scales trivially |

**Per-tenant rate limiting** is keyed `{provider}:{workspaceId}` from day one, so the transition requires no code change — only the credential source changes.

---

## 8. Cost scaling

| Stage | Workspaces | Infra/month | AI+image/month | Revenue @ £49 avg | Gross margin |
|---|---|---|---|---|---|
| 1 | 1 | £58 | £66 | — | — |
| 2 | 50 | £310 | £2,400 | £2,450 | ~-10% (investment phase) |
| 3 | 1,000 | £2,600 | £34,000 | £49,000 | ~25% |
| 3 + shared caches | 1,000 | £2,600 | £21,000 | £49,000 | **~52%** |
| 4 | 10,000 | £18,000 | £185,000 | £490,000 | ~58% |

**The single most important number in this table** is the effect of shared market-fact caching at Stage 3: gross margin roughly doubles. It is why the caching boundary described in §6 is an architectural priority rather than an optimisation, and why usage-based components in pricing (doc 21) are necessary to keep heavy users from destroying unit economics.

---

## 9. Performance under load

| Scenario | Design response |
|---|---|
| 200 concurrent runs | Queue absorbs; per-tenant fair scheduling prevents starvation; UI shows honest queue position and estimated start |
| A single 20-shop deep run with 3,000 listings | Batched vision, streamed statistics, chunked inserts; memory bounded by processing in fixed-size chunks, never loading the full set |
| Report page over 40k listings | Server-side aggregation, cursor pagination, virtualised table, pre-computed facets |
| Provider outage | Circuit breaker; runs pause at `blocked_external` rather than failing; automatic resume |
| Traffic spike | Web autoscales; workers autoscale on queue age; database protected by connection pooling and by the fact that heavy work is never in the request path |
| Runaway tenant spend | Budget guard denies before the call; workspace monthly cap pauses non-essential queues |

---

## 10. Data volume management

| Data | Growth driver | Control |
|---|---|---|
| Listing snapshots | Runs × shops × listings | 180-day raw retention, then Parquet; aggregates retained |
| Competitor images | Unique thumbnails | Content-hash dedupe; 180-day retention; thumbnails only (no full-size) |
| Artwork | Accepted designs × renditions | Retained indefinitely (it is the product); originals in infrequent-access tier after 90 days |
| Mockups | Published products × images | Retained while listing is live; archived 90 days after delisting |
| AI raw responses | Every call | 90 days, then purged; structured outputs retained |
| Provider request/response bodies | Every call | 30 days, then nulled |
| Vectors | Style profiles + concepts + artwork | Partition by workspace at Stage 3; quantisation if memory-bound |

---

## 11. Multi-region (Stage 4)

Only when latency or data residency demands it.

| Concern | Approach |
|---|---|
| Web | Edge-rendered, multi-region by default |
| Database | Primary in the workspace's home region; regional read replicas; workspace-to-region pinning stored on the workspace record |
| Object storage | Regional buckets with the workspace's region on the asset record |
| Queues | Regional clusters; jobs never cross regions |
| Residency | Workspace region is immutable after creation; migration is an explicit, supported operation with downtime |

---

## 12. Capacity planning and load testing

| Test | Trigger | Assertion |
|---|---|---|
| Sustained load (k6) | Nightly | p95 within budget at 3× current peak |
| Spike | Weekly | 10× burst for 60 s degrades gracefully, no errors, queue absorbs |
| Soak | Weekly, 4 h | No memory growth, no connection leak, no queue drift |
| Data volume | Monthly | Reports over a 10× seeded dataset meet latency budgets |
| Chaos | Weekly | Kill worker mid-run → identical output, zero duplicate side-effects |
| Provider degradation | Weekly | Injected 429/500/timeout → breakers open, runs pause, no data loss |

**Capacity triggers (act, don't watch):** queue age p95 > 2 min → add workers. DB CPU > 70% sustained → add a replica or optimise the top query by total time. Connection utilisation > 80% → PgBouncer. Storage growth > 20%/month → review retention. AI cost per run drifting > 15% above budget → prompt/tier review.

---

## 13. What we deliberately do not build early

| Deferred | Why |
|---|---|
| Microservices | Module boundaries already exist; distributed transactions and operational overhead would slow delivery for no current benefit. Split at Stage 4, along the existing package seams. |
| Kubernetes | Managed platforms carry Stage 1–3 comfortably. Adopt when multi-region or bespoke scheduling forces it. |
| Event sourcing / CQRS | Snapshot tables and the append-only run/step model already provide auditability and replay where it matters, without the read-model complexity. |
| Dedicated vector database | pgvector with HNSW handles millions of vectors. Move only when index memory, not query latency, is the binding constraint. |
| Data warehouse | Postgres replicas answer analytics through Stage 3. Introduce a columnar store when analytical queries begin to affect transactional latency. |
| Temporal | The custom orchestrator is ~600 lines and fits the domain exactly. Adopt Temporal at Stage 4 when cross-region durability and long-lived timers justify the operational cost — the DAG definition ports directly. |
