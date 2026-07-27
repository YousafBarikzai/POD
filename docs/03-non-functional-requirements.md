# 03 — Non-Functional Requirements

**Version:** 1.0

Every NFR is stated as a **measurable budget** with a **verification method**. An NFR without a number is an opinion.

---

## 1. Performance

### 1.1 Interactive latency (web tier)

| Operation | p50 | p95 | p99 | Verification |
|---|---|---|---|---|
| Page shell TTFB | 120 ms | 400 ms | 900 ms | Synthetic + RUM |
| First Contentful Paint | 0.9 s | 1.8 s | 3.0 s | Lighthouse CI in pipeline |
| Largest Contentful Paint | 1.4 s | 2.5 s | 4.0 s | RUM (web-vitals) |
| Interaction to Next Paint | 90 ms | 200 ms | 500 ms | RUM |
| Read query (tRPC, cached) | 25 ms | 90 ms | 250 ms | Server-side histogram |
| Read query (uncached, joined) | 90 ms | 350 ms | 800 ms | Server-side histogram |
| Write mutation (non-pipeline) | 60 ms | 250 ms | 600 ms | Server-side histogram |
| Report page with 3k listings (virtualised table) | 400 ms | 1.2 s | 2.5 s | Load test with seeded fixture |
| Progress stream event lag | 200 ms | 800 ms | 2 s | Client-side measurement |

**Budget rule:** no single tRPC procedure may exceed **8 database round trips** or **200 ms of DB time** at p95. Violations fail the performance test suite.

### 1.2 Pipeline latency (async tier)

| Step | Target p95 | Hard timeout |
|---|---|---|
| 1 Input validation & niche resolution | 2 s | 30 s |
| 2 Opportunity report | 90 s | 6 min |
| 3 Competitor collection (10 shops) | 150 s | 15 min |
| 3c Visual style extraction (300 listings) | 120 s | 10 min |
| 4 Success analysis | 45 s | 5 min |
| 5 Failure analysis | 40 s | 5 min |
| 6 Gap analysis | 50 s | 5 min |
| 7 Prediction (20 concepts) | 15 s | 3 min |
| 8 Concept generation (20) | 70 s | 6 min |
| 9 Legal screening (per concept) | 12 s | 3 min |
| 10 Artwork (4 variants + post-processing) | 180 s | 12 min |
| 11 Product recommendation | 20 s | 3 min |
| 12 SEO (10 variations) | 60 s | 5 min |
| 13 Printify product + mockups | 120 s | 10 min |
| 14 Etsy draft + images | 60 s | 6 min |
| **Steps 1–8 end-to-end (standard depth)** | **≤ 11 min** | 40 min |

**Perceived latency requirement:** the operator must see meaningful partial output within **20 seconds** of starting a run (niche resolution + sub-niche candidates stream first), and every subsequent step must render its result the moment it completes rather than waiting for the run to finish.

### 1.3 Throughput

| Dimension | Phase 1 target | Stage-3 SaaS target |
|---|---|---|
| Concurrent research runs | 3 | 200 |
| Listing snapshots ingested/hour | 20,000 | 2,000,000 |
| Vision analyses/hour | 3,000 | 150,000 |
| Artwork generations/hour | 60 | 4,000 |
| Etsy drafts created/hour | 30 | 1,500 (across tenants) |

---

## 2. Reliability & availability

| Metric | Target |
|---|---|
| Web tier availability (monthly) | 99.5% Phase 1 → 99.9% SaaS |
| Worker tier availability | 99.0% Phase 1 → 99.9% SaaS |
| Successful run completion rate (excluding operator cancellation) | ≥ 97% |
| Data durability | 99.999999999% (object storage), Postgres with PITR |
| **RPO** | ≤ 5 minutes |
| **RTO** | ≤ 2 hours Phase 1, ≤ 30 minutes SaaS |
| Max acceptable duplicate side-effects (Etsy listings, Printify products, paid generations) | **Zero** |

### 2.1 Failure semantics

- **At-least-once delivery** for all queue jobs; **exactly-once effect** enforced by idempotency keys on every side-effecting operation.
- Every step is **retriable** and **resumable**; a step's output is written transactionally with its status transition.
- **Partial success is a first-class state.** No run fails wholesale because one shop, one image, or one registry lookup failed.
- **Circuit breakers** per external provider: 5 consecutive failures or > 50% error rate over 20 calls opens the breaker for 60 s (exponential to 15 min), during which dependent steps are marked `blocked_external` rather than failing.
- **Poison-message handling:** a job failing 5 times moves to a dead-letter queue with full context and raises an alert; it never blocks the queue.

### 2.2 Data integrity

- All multi-row state transitions run inside a transaction.
- Financial and cost figures are append-only ledgers, never mutated.
- Snapshots (listing, performance) are immutable — corrections are new rows.
- Foreign keys enforced everywhere; no soft-referential integrity.
- A nightly consistency job verifies: every artwork has a reachable asset, every published listing has an `etsy_listing_id`, every run's step statuses are consistent with its own status.

---

## 3. Scalability

| Requirement | Detail |
|---|---|
| NFR-S1 | The system **MUST** scale horizontally on the web tier (stateless, session in signed cookie + Redis). |
| NFR-S2 | The worker tier **MUST** scale by adding processes; queue concurrency is configurable per queue. |
| NFR-S3 | No component may hold run state in process memory across steps. |
| NFR-S4 | The database schema **MUST** support partitioning of the highest-volume tables (`listing_snapshots`, `performance_snapshots`, `provider_calls`, `ai_calls`) by time from day one — declared as partitioned tables even when there is a single partition. |
| NFR-S5 | All list endpoints **MUST** be cursor-paginated. Offset pagination is prohibited on tables that can exceed 10k rows. |
| NFR-S6 | Any operation touching > 1,000 rows **MUST** be batched and run in a worker, never in a request. |
| NFR-S7 | Per-tenant resource isolation **MUST** be achievable via queue partitioning and per-workspace rate limits without schema change. |

Growth targets and stage-by-stage plan in [doc 16](16-scalability-architecture.md).

---

## 4. Cost & efficiency

| Budget | Limit |
|---|---|
| Full research run (standard depth), AI + data | ≤ £1.20 |
| Deep run | ≤ £2.60 |
| Artwork per accepted design (incl. retries) | ≤ £0.60 |
| Published product, all-in marginal cost | ≤ £1.80 |
| Monthly infrastructure, single user | ≤ £140 |
| AI spend as % of total marginal cost | ≤ 65% |

**Enforcement mechanisms (all mandatory):**
1. Per-run budget with a hard stop (FR-010).
2. Model tiering — cheapest capable tier per task (doc 10 §3).
3. Prompt caching for all repeated system/context blocks.
4. Content-hash caching for vision analysis, registry lookups, embeddings, and blueprint catalogues.
5. Aggregation-before-prompting: statistical summaries, never raw record dumps, into LLM context.
6. Concept-before-artwork gating so image spend only follows human approval.
7. A monthly workspace cap that pauses non-essential jobs when exceeded.

---

## 5. Security

Full model in [doc 15](15-security-architecture.md). Non-negotiable NFRs:

| ID | Requirement |
|---|---|
| NFR-SEC1 | All traffic over TLS 1.3; HSTS with preload; no mixed content. |
| NFR-SEC2 | Secrets never in source, logs, error messages, or client bundles. Verified by CI secret scanning. |
| NFR-SEC3 | Third-party credentials encrypted at rest with envelope encryption (AES-256-GCM, KMS-held master key), decrypted only in worker/API memory, never sent to the client. |
| NFR-SEC4 | Authentication: password (Argon2id, ≥ 64 MB memory cost) + mandatory TOTP second factor. |
| NFR-SEC5 | Session cookies: `HttpOnly`, `Secure`, `SameSite=Lax`, rotating on privilege change, absolute expiry 30 days, idle expiry 7 days. |
| NFR-SEC6 | Every query is workspace-scoped at the data-access layer; Postgres RLS policies are enabled and enforced in the SaaS phase and written from day one. |
| NFR-SEC7 | All external content (competitor titles, descriptions, tags, registry results) entering an LLM prompt **MUST** be treated as untrusted and delimited/escaped; no external content may alter instructions. |
| NFR-SEC8 | AI-generated output **MUST NOT** be executed, rendered as HTML, or used to construct queries without validation and sanitisation. |
| NFR-SEC9 | Full audit log of authentication, credential changes, publish actions, legal overrides, and config activations. Immutable, 7-year retention. |
| NFR-SEC10 | Dependency scanning and SBOM generation in CI; critical CVEs block deploy. |
| NFR-SEC11 | Rate limiting on all authenticated endpoints and strict limits on unauthenticated ones. |
| NFR-SEC12 | CSP with nonce-based script policy; no `unsafe-inline` or `unsafe-eval` in production. |

---

## 6. Privacy & compliance

| ID | Requirement |
|---|---|
| NFR-P1 | Only publicly available competitor shop/listing data is stored. **No personal data of buyers or of competitor shop owners beyond the public shop name is collected.** Reviewer names and review text are stored in aggregate/statistical form only, never as identifiable records. |
| NFR-P2 | Data acquisition **MUST** respect provider terms; the compliance posture and per-adapter legal assessment is documented and reviewed each phase (doc 11 §7). |
| NFR-P3 | UK GDPR/EU GDPR readiness: lawful basis documented, data inventory maintained, export and erasure implemented (FR-1705), DPA required with every subprocessor before SaaS launch. |
| NFR-P4 | Subprocessor register maintained (Anthropic, Ideogram, Etsy, Printify, hosting, storage, monitoring). |
| NFR-P5 | Retention: raw competitor snapshots 180 days (aggregates retained indefinitely); provider request/response bodies 30 days; AI raw responses 90 days; audit logs 7 years; owned entities indefinite until deletion. |
| NFR-P6 | No customer data is used to train third-party models; zero-data-retention or no-training terms confirmed with each AI provider and recorded. |
| NFR-P7 | Cookie use limited to strictly necessary in Phase 1; no third-party analytics that set cookies without consent. |

---

## 7. Observability

| ID | Requirement |
|---|---|
| NFR-O1 | Structured JSON logs with a mandatory field set: `timestamp`, `level`, `service`, `env`, `trace_id`, `span_id`, `workspace_id`, `user_id`, `run_id`, `step_id`, `event`, `duration_ms`. |
| NFR-O2 | Distributed tracing via OpenTelemetry across web → API → queue → worker → external provider. A run's full trace must be viewable as one waterfall. |
| NFR-O3 | Metrics (RED + USE): request rate/errors/duration per procedure; queue depth, job age, processing time, failure rate per queue; provider latency, error rate, cost, and quota consumption; per-step success rate and duration. |
| NFR-O4 | Error tracking with source maps, release tagging, and user/workspace context (no secrets, no PII). |
| NFR-O5 | Cost telemetry is a first-class metric stream: tokens, images, provider calls, and currency cost per run, per step, per workspace. |
| NFR-O6 | Alerts with defined severity and runbook links: run failure rate > 5% (15 min window), queue age > 10 min, provider breaker open > 5 min, budget 90% consumed, DB connection saturation > 80%, error rate > 1%, publish failure, legal-engine failure. |
| NFR-O7 | Every alert **MUST** link to a runbook in `docs/runbooks/`. An alert without a runbook is a defect. |
| NFR-O8 | Log retention: 30 days hot, 12 months cold for audit-relevant events. |

---

## 8. Maintainability & quality

| ID | Requirement |
|---|---|
| NFR-M1 | TypeScript `strict` everywhere; `any` prohibited outside adapter boundary parsing, where it must be immediately narrowed by a schema. |
| NFR-M2 | Every external payload is parsed through a schema (Zod) at the boundary; internal code never sees unvalidated shapes. |
| NFR-M3 | Unit test coverage ≥ 80% on `packages/domain` (scoring, statistics, pricing, validators) — these are pure functions and must be exhaustively tested, including property-based tests for scoring monotonicity and bounds. |
| NFR-M4 | Integration tests for every adapter against recorded fixtures (VCR-style); no test may hit a live external API in CI. |
| NFR-M5 | End-to-end tests (Playwright) for the five critical journeys: run a research pipeline; select concepts; generate artwork; create an Etsy draft; publish from Final Review (against sandbox/mocks). |
| NFR-M6 | Database migrations are forward-only, reviewed, and tested against a production-shaped seed. Every migration must be backwards-compatible with the previously deployed application version (expand/contract). |
| NFR-M7 | Cyclomatic complexity ceiling of 15 per function; enforced by lint. |
| NFR-M8 | No cross-package imports that bypass the declared public entrypoint of a package. |
| NFR-M9 | ADRs required for any decision affecting more than one package, data durability, cost, or an external contract. |
| NFR-M10 | All prompts are versioned files under source control with tests; prompts are never inline string literals. |

---

## 9. Usability & accessibility

| ID | Requirement |
|---|---|
| NFR-U1 | WCAG 2.2 AA compliance: contrast ≥ 4.5:1 for text, full keyboard navigability, visible focus, semantic landmarks, correct ARIA on custom widgets. |
| NFR-U2 | Every long-running action shows determinate progress with step names and elapsed/estimated time. |
| NFR-U3 | Every error message states what happened, why, and the next action. Raw provider errors are never shown unmapped. |
| NFR-U4 | Every score in the UI is accompanied by an accessible explanation (tooltip + expandable evidence panel). |
| NFR-U5 | Destructive and spend-incurring actions require confirmation showing the cost/consequence. |
| NFR-U6 | Colour is never the sole carrier of meaning in charts or status indicators. |
| NFR-U7 | The app is usable at 1280×800 minimum; data-dense tables degrade gracefully to horizontal scroll containers. Tablet-responsive; native mobile is out of scope. |
| NFR-U8 | Dark and light themes, both meeting contrast requirements. |

---

## 10. Portability & vendor risk

| ID | Requirement |
|---|---|
| NFR-V1 | No hosting-vendor-proprietary runtime API in application code. Everything runs in a plain Node process and a container. |
| NFR-V2 | Object storage accessed via the S3 API only, so R2/S3/MinIO are interchangeable. |
| NFR-V3 | Queue access via a thin interface so BullMQ can be replaced (SQS, Temporal) without touching domain code. |
| NFR-V4 | The AI provider is behind an interface with model-tier semantics, not model names, so tiers can be remapped by configuration. |
| NFR-V5 | The image generation provider is behind an interface; Ideogram is the default implementation, not a hard dependency. |
| NFR-V6 | The market data provider is a chain of adapters with a guaranteed offline path (CSV import). |
| NFR-V7 | All data is exportable in open formats (JSON/CSV/PNG/SVG). |

---

## 11. Internationalisation (readiness, not implementation)

| ID | Requirement |
|---|---|
| NFR-I1 | All user-facing strings live in a message catalogue from day one, even though only `en-GB` is shipped. |
| NFR-I2 | Currency, number and date formatting via `Intl`, never hand-rolled. |
| NFR-I3 | The schema supports multi-currency (amount + currency per row) and multi-locale listings (locale on listing content) without migration. |
| NFR-I4 | Text length validators are configurable per marketplace locale. |

---

## 12. Operational

| ID | Requirement |
|---|---|
| NFR-OP1 | Zero-downtime deploys; rolling worker restarts that drain in-flight jobs (grace period 60 s). |
| NFR-OP2 | Feature flags for every new engine so partial functionality ships dark. |
| NFR-OP3 | One-command local environment (`docker compose up` + `pnpm dev`) with seeded fixture data and mocked providers. |
| NFR-OP4 | Backups: continuous WAL archiving with PITR; nightly logical dump retained 30 days; object storage versioning enabled. **Restore is tested monthly** and the test result recorded. |
| NFR-OP5 | Runbooks for: provider outage, budget exhaustion, stuck run, Etsy token revocation, DB failover, queue backlog, bad scoring config rollback. |
| NFR-OP6 | A maintenance mode that pauses queues, drains gracefully and shows an informative UI. |
| NFR-OP7 | Configuration is environment-variable driven and validated at boot; the process refuses to start on invalid config rather than failing at runtime. |

---

## 13. NFR verification matrix

| NFR area | How verified | When |
|---|---|---|
| Interactive latency | Lighthouse CI + k6 load test | Every PR (budget check), nightly full run |
| Pipeline latency | Synthetic run against fixture providers | Nightly |
| Reliability/idempotency | Chaos test: kill worker mid-step, assert no duplicates | Weekly in staging |
| Cost budgets | Cost telemetry assertions in synthetic run | Nightly |
| Security | SAST, dependency scan, secret scan, quarterly manual review | Every PR + quarterly |
| Accessibility | axe-core in E2E, manual keyboard pass | Every PR + per release |
| Backups | Automated restore drill into scratch DB | Monthly |
| Data integrity | Nightly consistency job | Nightly |
