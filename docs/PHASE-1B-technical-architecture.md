# POD Intelligence — Phase 1B

# Technical Architecture Specification

**Document type:** Technical Architecture Specification
**Phase:** 1B — *how the software will be built*
**Version:** 1.0
**Status:** For engineering review
**Prerequisite:** [Part 1 — Product Definition](PART-1-product-definition.md)

> **Scope note.** This document specifies architecture, technology choices, data structures, service boundaries, interfaces and deployment. **It contains no application code.** Where structures are shown, they are shown as design tables and contracts, not as implementations. Implementation begins in Phase 2.

---

## Contents

| Part | Section |
|---|---|
| **A** | [System Architecture](#part-a--system-architecture) |
| **B** | [Technology Decisions](#part-b--technology-decisions) |
| **C** | [Database Design](#part-c--database-design) |
| **D** | [API Design](#part-d--api-design) |
| **E** | [Backend Services Architecture](#part-e--backend-services-architecture) |
| **F** | [AI Architecture](#part-f--ai-architecture) |
| **G** | [Integration Architecture](#part-g--integration-architecture) |
| **H** | [Background Job System](#part-h--background-job-system) |
| **I** | [Security Architecture](#part-i--security-architecture) |
| **J** | [Scalability Architecture](#part-j--scalability-architecture) |
| **K** | [Project Structure](#part-k--project-structure) |
| **L** | [Deployment Architecture](#part-l--deployment-architecture) |

---

# Part A — System Architecture

## A.1 Architectural style

**A modular monolith with a separate worker tier.**

Not microservices. Not a single process doing everything. The reasoning:

| Consideration | Conclusion |
|---|---|
| Team size | One to three engineers. Microservices impose a coordination and operational tax that only pays back at larger team sizes. |
| Workload shape | Two genuinely different profiles — sub-second request serving, and 3–22 minute CPU-and-network-heavy pipelines. These **must** be separated or interactive latency becomes unpredictable. |
| Transactional needs | Run state, cost ledgers and step outputs must be written atomically. Distributed transactions across services would be a self-inflicted wound. |
| Future splitting | Module boundaries are drawn now exactly where service boundaries would go later. Splitting becomes an extraction, not a rewrite. |

So: **one codebase, clear internal module boundaries, three deployable processes.**

```
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│   WEB APP    │   │    WORKER    │   │  SCHEDULER   │
│  (stateless) │   │  (stateless) │   │ (singleton)  │
│   N replicas │   │   N replicas │   │  1 + failover│
└──────────────┘   └──────────────┘   └──────────────┘
```

**Why the worker is separate.** A research run consumes 100% of a core during statistical analysis and image processing for minutes at a time. Colocating that with request serving would make page loads unpredictable and would make it impossible to scale the two workloads independently. This is the single most important topology decision.

**Why the scheduler is separate.** Cron work (daily performance sync, catalogue sync, retention, consistency checks) needs exactly-once semantics. A tiny dedicated process holding a distributed lock is simpler and far more debuggable than leader election distributed across a worker fleet.

---

## A.2 High-level system diagram

```
                                    ┌─────────────┐
                                    │   BROWSER   │
                                    │  (operator) │
                                    └──────┬──────┘
                                           │ HTTPS
                                           ▼
                    ╔══════════════════════════════════════════╗
                    ║              EDGE / CDN                  ║
                    ║   TLS · WAF · static assets · caching    ║
                    ╚══════════════════┬═══════════════════════╝
                                       │
        ┌──────────────────────────────┴──────────────────────────────┐
        │                     WEB APPLICATION TIER                     │
        │                    (stateless, N replicas)                   │
        │  ┌────────────────┐ ┌──────────────┐ ┌──────────────────┐  │
        │  │ Server-rendered│ │  Typed API   │ │  Progress stream │  │
        │  │      pages     │ │   handler    │ │      (SSE)       │  │
        │  └────────────────┘ └──────────────┘ └──────────────────┘  │
        │  ┌────────────────┐ ┌──────────────┐ ┌──────────────────┐  │
        │  │  Auth / session│ │ Webhook recv │ │  OAuth callbacks │  │
        │  └────────────────┘ └──────────────┘ └──────────────────┘  │
        └───────┬──────────────────┬───────────────────┬──────────────┘
                │                  │                   │
        reads/writes          enqueue jobs        subscribe events
                │                  │                   │
                ▼                  ▼                   ▼
        ┌───────────────┐  ┌──────────────────────────────────┐
        │  POSTGRESQL   │  │             REDIS                │
        │               │  │  queues · cache · rate limits ·  │
        │ • relational  │  │  locks · sessions · pub-sub      │
        │ • time-series │  └───────┬──────────────────────────┘
        │   partitions  │          │ consume
        │ • full-text   │          ▼
        │ • vectors     │  ┌──────────────────────────────────────────┐
        │ • JSON        │  │            WORKER TIER                   │
        └───────▲───────┘  │        (stateless, N replicas)           │
                │          │  ┌────────────────────────────────────┐  │
                │          │  │      RUN ORCHESTRATOR              │  │
                └──────────┼──┤  durable step state machine        │  │
                           │  │  budget guard · retries · resume   │  │
                           │  └────────────┬───────────────────────┘  │
                           │               │ invokes                   │
                           │  ┌────────────▼───────────────────────┐  │
                           │  │        DOMAIN SERVICES             │  │
                           │  │ research · competitor · style ·    │  │
                           │  │ success · failure · gap · predict ·│  │
                           │  │ concept · legal · artwork ·        │  │
                           │  │ product · seo · publish · learn    │  │
                           │  └────────────┬───────────────────────┘  │
                           └───────────────┼──────────────────────────┘
                                           │ via adapters only
        ┌──────────────────────────────────┼──────────────────────────────┐
        │                        ADAPTER LAYER                             │
        │  every external call passes through here: schema validation,     │
        │  retry, rate limiting, circuit breaking, cost recording, tracing │
        └───┬─────────┬─────────┬─────────┬─────────┬─────────┬───────────┘
            │         │         │         │         │         │
            ▼         ▼         ▼         ▼         ▼         ▼
        ┌───────┐ ┌───────┐ ┌────────┐ ┌──────┐ ┌────────┐ ┌──────────┐
        │ CLAUDE│ │IDEOGRAM│ │  ETSY  │ │PRINT-│ │ MARKET │ │TRADEMARK │
        │  AI   │ │ images │ │  API   │ │ IFY  │ │  DATA  │ │REGISTRIES│
        └───────┘ └───────┘ └────────┘ └──────┘ └────────┘ └──────────┘

                    ┌──────────────────────────────┐
                    │      OBJECT STORAGE          │
                    │  artwork · mockups ·         │
                    │  competitor thumbnails ·     │
                    │  reports · imports · exports │
                    └──────────────────────────────┘
                            ▲              ▲
                            │              │
                    web (signed URLs)   worker (writes)

                    ┌──────────────────────────────┐
                    │        SCHEDULER             │
                    │  performance sync · catalogue│
                    │  sync · retention · consist- │
                    │  ency · recalibration        │
                    └──────────────┬───────────────┘
                                   │ enqueues into Redis
                                   ▼
                            (worker tier)

        ┌───────────────────────────────────────────────────────────┐
        │                     OBSERVABILITY                         │
        │  structured logs · distributed traces · metrics · errors  │
        │  · cost telemetry — emitted by all three tiers            │
        └───────────────────────────────────────────────────────────┘
```

---

## A.3 Frontend architecture

### Structure

**A server-first React application.** Pages render on the server by default; interactivity is added only where it is genuinely needed, pushed to the leaves of the component tree.

| Layer | Responsibility |
|---|---|
| **Route shell** | Navigation, layout, theme, auth context, global providers |
| **Server components** | Initial data fetch and render for reports, tables, dashboards. No client JavaScript shipped for these. |
| **Client components** | Interactive islands: filters, selection, editors, charts, live progress |
| **Server-state layer** | Query caching, background refetch, targeted invalidation |
| **Client-state layer** | Wizard progress, selection sets, drawer open/closed — nothing else |
| **URL as state** | Filters, sort, tab, pagination live in the URL so every view is shareable and restorable |

### Why server-first matters here

This product's dominant screens are **data-dense reports over thousands of rows**. Shipping that data to the browser as JSON and rendering it client-side would mean large payloads, slow first paint and heavy memory use. Rendering on the server and streaming sections independently means:

- The opportunity score appears while the competitor table is still being prepared.
- A slow section never blocks the page.
- Payloads are HTML, not duplicated JSON-plus-markup.

### Live progress

A research run takes minutes. The frontend must show meaningful progress, not a spinner.

```
Browser ──── opens SSE connection ────► Web tier
                                            │ subscribes
                                            ▼
                                       Redis pub/sub
                                            ▲
                                            │ publishes
Worker ──── emits step events ──────────────┘
```

- **Server-Sent Events**, not WebSockets. Progress is one-directional; SSE is simpler, survives proxies better, and reconnects natively with resume-from-last-event.
- Events carry an ID so a dropped connection resumes rather than restarts.
- Heartbeat every 15 seconds to keep intermediaries from closing the connection.
- If SSE fails twice, the client falls back to polling. Progress must never simply stop.
- Events are coalesced server-side to at most two per second per step, so a chatty worker cannot flood the browser.

### Rendering strategy per screen type

| Screen | Strategy | Reason |
|---|---|---|
| Dashboard | Server-rendered with streaming sections | Mixed data sources, some slow |
| Reports | Server-rendered, virtualised tables, client-side filtering on a server-paginated set | Large data, heavy interaction |
| Run monitor | Client component with SSE | Inherently live |
| Concept board | Server-rendered shell, client selection state | Interactive but not live |
| Artwork studio | Client-heavy | Canvas manipulation, comparison |
| Settings | Server-rendered forms | Simple, mostly static |

---

## A.4 Backend architecture

### Layered, with strictly enforced direction

```
    ┌─────────────────────────────────────────────────────┐
    │  ENTRY POINTS                                       │
    │  API handlers · job processors · webhook receivers  │
    │  · scheduled tasks                                  │
    │  — thin. Validate, delegate, serialise. Nothing else│
    └────────────────────────┬────────────────────────────┘
                             │
    ┌────────────────────────▼────────────────────────────┐
    │  ORCHESTRATION                                      │
    │  Run orchestrator · use-case coordinators           │
    │  — sequencing, transactions, budget, human gates    │
    └────────────────────────┬────────────────────────────┘
                             │
    ┌────────────────────────▼────────────────────────────┐
    │  DOMAIN SERVICES (the 13 engines)                   │
    │  — fetch inputs, invoke pure logic, persist outputs │
    └───────┬──────────────────────────────┬──────────────┘
            │                              │
    ┌───────▼──────────┐        ┌──────────▼──────────────┐
    │  PURE DOMAIN     │        │  INFRASTRUCTURE         │
    │  scoring · stats │        │  repositories · queue · │
    │  pricing · rules │        │  adapters · storage     │
    │  validators      │        │                         │
    │  — NO I/O AT ALL │        │                         │
    └──────────────────┘        └─────────────────────────┘
```

### The rules that make this hold

These are enforced by automated dependency analysis in the build, not by convention:

1. **The pure domain layer imports nothing but shared types.** No database, no network, no clock, no randomness. Every function is deterministic and synchronously testable.
2. **Engines never import other engines.** They communicate through the orchestrator and the database. This is what makes them independently extractable later.
3. **Nothing outside the adapter layer imports a vendor SDK.**
4. **Nothing outside the repository layer talks to the database.**
5. **Entry points contain no business logic.** If a handler exceeds roughly ten lines, logic has leaked into the wrong layer.

**Why this matters more than usual here.** The product's core value is a set of scoring and statistical calculations that must be reproducible, auditable and improvable over time. Keeping those in a pure, I/O-free layer is what makes them testable to the standard the product requires — including property-based testing for monotonicity and bounds.

---

## A.5 Database architecture

**A single PostgreSQL database serving five distinct roles.**

| Role | Mechanism | Why not a separate system |
|---|---|---|
| Relational store | Native tables, foreign keys | — |
| Time-series | Declarative monthly range partitioning | Snapshots must join relationally to their parent entities |
| Full-text search | Generated search vectors, trigram indexes | Search is navigational over tens of thousands of rows, not relevance-critical over millions |
| Vector similarity | Vector extension with approximate-nearest-neighbour indexes | Similarity queries must combine with relational filters (workspace, run, date) in one query |
| Flexible payloads | JSON columns for provider responses and feature vectors | Schema-on-read is correct for data whose shape is owned by someone else |

**Read/write separation is explicit from day one.** Repository methods declare whether they read or write; reads route to a replica when one exists. Routing is never inferred, because replica lag silently breaking a read-after-write is a class of bug that is extremely hard to diagnose.

**Partitioning is declared from the first migration**, even with a single partition. Adding partitioning to a billion-row table later is an outage. Adding it now is one line in a migration.

---

## A.6 AI service architecture

AI is not a library call scattered through the codebase. It is a **service layer with a uniform contract**.

```
        Engine
          │  requests { purpose, prompt id + version, variables,
          │             output contract, untrusted inputs, cache policy }
          ▼
   ┌─────────────────────────────────────────────────────┐
   │              AI SERVICE LAYER                       │
   │                                                     │
   │  1. Resolve capability tier → concrete model        │
   │  2. Render prompt from the versioned registry       │
   │  3. Quarantine untrusted external text              │
   │  4. Check cache by input hash                       │
   │  5. Reserve budget                                  │
   │  6. Acquire rate-limit slot                         │
   │  7. Call provider with structured-output contract   │
   │  8. Validate against contract → repair → retry      │
   │  9. Record tokens, cost, latency, prompt version    │
   │ 10. Cache if the purpose is deterministic           │
   └─────────────────────────────────────────────────────┘
          │
          ▼
     Validated, typed result — or an explicit failure
```

**Four capability tiers**, referenced by role rather than by model name so that model upgrades are configuration changes:

| Tier | Used for |
|---|---|
| **Reasoning** | Concept generation, artwork briefs, copyright risk assessment, gap explanations |
| **Analysis** | Sub-niche discovery, SEO generation, entity extraction, safer alternatives |
| **Extraction** | Simple classification and structured transforms |
| **Vision** | Competitor image analysis, artwork safety review |

Full detail in [Part F](#part-f--ai-architecture).

---

## A.7 External integration architecture

**Every external dependency sits behind an adapter with an identical contract.**

```
   Engine  ──►  Adapter interface  ──►  Provider client  ──►  External API
                      │
                      ├── response schema validation
                      ├── retry with backoff and jitter
                      ├── distributed rate limiting
                      ├── circuit breaking
                      ├── timeout enforcement
                      ├── cost recording
                      ├── error translation
                      └── request/response logging
```

**Six adapter families:**

| Adapter | Volatility | Fallback if it fails |
|---|---|---|
| AI provider | Model changes are frequent; API is stable | Tier remapping via configuration |
| Image generation | Young product; pricing and terms volatile | Interface allows substitution; ~80% of the artwork pipeline is provider-independent |
| Etsy | Stable API, evolving policy | None — it is the marketplace. Failures pause work rather than losing it. |
| Printify | Moderately stable | Publish-bridge mode as an alternative path |
| Market data | **No public API. Highest volatility in the system.** | A chain of adapters with a guaranteed operator-supplied-file path that cannot break |
| Trademark registries | Stable but heterogeneous | Cached results; degraded screening with explicit warning |

The market data adapter chain is the most important structural decision in the integration layer and is specified in [Part G](#part-g--integration-architecture).

---

## A.8 Storage architecture

**Object storage for bytes, database for facts about bytes.**

| Bucket | Contents | Lifecycle |
|---|---|---|
| `artwork` | Generated originals and every derived rendition | Retained indefinitely — it is the product |
| `mockups` | Product images from the fulfilment provider | Retained while the listing is live, archived 90 days after delisting |
| `market` | Competitor thumbnails used for visual analysis | 180 days, then garbage-collected if unreferenced |
| `documents` | Report exports, data exports, imported files | 90 days for exports, indefinite for imports |
| `system` | Raw AI responses for audit | 90 days |

**Principles**

1. **Content-addressed.** Every object key derives from the SHA-256 of its content. The same image is never stored, downloaded or analysed twice — the single largest cost saving in the visual analysis pipeline.
2. **Private by default.** No public access. Everything is served through short-lived signed URLs.
3. **Immutable.** Processing never overwrites. Background removal produces a new object; upscaling produces another. The lineage is recorded as a tree.
4. **Database holds metadata only.** Dimensions, format, size, hash, purpose, lineage. Bytes never enter the database.
5. **Versioning enabled.** An accidental deletion is recoverable.

**Derivation tree per artwork:**

```
original (provider output, never modified)
   └── background removed
         └── auto-cropped
               └── upscaled to print resolution  ──► print file
                     ├── web preview
                     ├── thumbnail
                     ├── transparency proof
                     └── vector version (where applicable)
```

---

## A.9 Authentication architecture

**Single user now, complete mechanism now.**

```
   Browser
     │  email + password
     ▼
   Password verification (memory-hard hashing)
     │
     ▼
   Second factor (time-based one-time password) — mandatory
     │
     ▼
   Session created: opaque random token
     │  ├─ token hash stored server-side with expiry and metadata
     │  └─ token returned in a hardened cookie
     ▼
   Every subsequent request:
     ├─ cookie → session lookup → user + workspace resolved server-side
     ├─ workspace scope injected into the data-access context
     └─ never trusted from client input
```

**Deliberate choices**

| Choice | Reason |
|---|---|
| Opaque session tokens, not self-contained tokens | Instant revocation. A stolen token can be killed immediately, which a self-contained token cannot. |
| Mandatory second factor | The account controls write access to a live revenue-generating shop. |
| Workspace resolved server-side, always | Eliminates an entire class of tenant-isolation bugs before multi-user even exists. |
| Step-up re-authentication | Required for credential changes, legal overrides, budget increases and disconnecting integrations. |
| Roles defined but unused | One `owner` row today. Enabling multi-user later adds rows, not schema. |

**Third-party credentials are a separate concern from user authentication** and are handled by envelope encryption, described in [Part I](#part-i--security-architecture).

---

## A.10 Background processing architecture

**The most important architectural component in the system.**

A research run spans roughly fourteen steps, takes 3–22 minutes, calls five external providers, spends real money, and includes a step that waits indefinitely for a human decision. It must survive process death without losing work or re-spending.

```
   ┌──────────────────────────────────────────────────────┐
   │              RUN (persisted aggregate)               │
   │  status · budget · spend · configuration · lineage   │
   └────────────────────────┬─────────────────────────────┘
                            │ has many
   ┌────────────────────────▼─────────────────────────────┐
   │              RUN STEPS (persisted)                   │
   │  key · order · dependencies · status · attempt ·     │
   │  input hash · output pointer · cost · timing · error │
   └──────────────────────────────────────────────────────┘

   Execution loop:
     claim step (skip-locked) → verify dependencies satisfied
       → reserve budget → mark running → execute
       → persist output + status + cost in ONE transaction
       → publish progress event → enqueue newly unblocked steps
```

**Properties this buys:**

| Property | How |
|---|---|
| **Durable** | Every transition is a committed database write |
| **Resumable** | A reaper detects steps whose worker died and requeues them |
| **Idempotent** | Step outputs are keyed by run and step; external effects carry their own idempotency keys |
| **Parallel** | Any step whose dependencies are satisfied is enqueued immediately |
| **Bounded** | Budget reserved before execution, reconciled after |
| **Observable** | Every transition emits an event and a metric |
| **Interruptible** | Cancellation is checked before every external call |
| **Human-gated** | Waiting for a decision is a run *status*, not a blocked job |

The step graph is **declarative data, not control flow**. Adding a step is a data change plus a handler — no restructuring.

---

## A.11 Deployment architecture summary

| Tier | Deployment | Scaling trigger |
|---|---|---|
| Web | Platform-managed, atomic deploys | Request latency or CPU |
| Worker | Containers, rolling restart with drain | Queue age |
| Scheduler | Single container with lock-based failover | Never scales; only fails over |
| Database | Managed PostgreSQL with point-in-time recovery | Vertical, then read replicas |
| Cache/queue | Managed Redis | Vertical |
| Object storage | S3-compatible service | Automatic |

Full detail in [Part L](#part-l--deployment-architecture).

---

## A.12 How every part communicates

| From | To | Mechanism | Notes |
|---|---|---|---|
| Browser | Web tier | HTTPS, typed API calls | Session cookie, CSRF protection on mutations |
| Browser | Web tier | Server-Sent Events | Live run progress, resumable |
| Browser | Object storage | Direct via signed URL | Bytes never proxied through the application |
| Web tier | Database | Connection pool, workspace scope set per checkout | Reads may route to a replica |
| Web tier | Redis | Job enqueue, cache, rate limits, session store | |
| Web tier | Worker | **Never directly.** Only via the queue | Complete decoupling |
| Worker | Database | Connection pool | Transactional step writes |
| Worker | Redis | Job consumption, locks, progress publication | |
| Worker | Object storage | Direct writes | |
| Worker | External APIs | **Only through adapters** | Rate-limited, circuit-broken, cost-recorded |
| Worker | Web tier | **Never directly.** Progress via Redis pub/sub | |
| Scheduler | Redis | Enqueues scheduled jobs | Holds a leader lock |
| External providers | Web tier | Inbound webhooks | Signature-verified, persisted then queued |
| All tiers | Observability | Structured logs, traces, metrics | Correlated by trace and run identifiers |

**Two rules worth stating explicitly:**

1. **The web tier and worker tier never call each other.** All coordination is through the database and the queue. Either can be restarted, scaled or deployed independently at any time.
2. **Webhook handlers do no work.** They verify the signature, persist the raw event, enqueue a job and return within milliseconds. Doing work inside a webhook request is how integrations start timing out and retrying, which compounds the problem.

---

# Part B — Technology Decisions

Each choice states what was selected, what was rejected, and why — with the reasoning grounded in *this* product's specific demands rather than general preference.

## B.1 Frontend

### Framework — **Next.js (App Router)**

| Alternative | Why not |
|---|---|
| Plain React SPA | Would ship megabytes of report data as JSON. First paint on a 3,000-row competitor table would be poor. |
| Remix | Genuinely good fit; loses on ecosystem depth for data-table and chart libraries, and on server-component granularity. |
| SvelteKit / Nuxt | Smaller ecosystem for the specific components this product leans on hardest. |
| Separate SPA + separate API | Doubles deployment surface and introduces API versioning between two things that always deploy together. |

**Why Next.js specifically fits this product:**
- Server components let report pages render on the server and stream section by section — critical when one section depends on a slow aggregate.
- Colocating the API with the frontend removes a whole category of CORS, versioning and deployment-skew problems for a product with exactly one client.
- Mature streaming and Suspense support, which the run monitor and reports need.

### Language — **TypeScript, strict mode**

Non-negotiable. This product has roughly sixty distinct data shapes flowing between analysis, scoring, generation and publishing. Type errors caught at build time are the cheapest bugs in the system. `any` is prohibited outside adapter-boundary parsing, where it must be narrowed immediately by a schema.

### UI library — **Tailwind CSS + Radix-based component primitives (shadcn/ui pattern)**

| Alternative | Why not |
|---|---|
| Material UI / Ant Design | Opinionated visual language that fights a custom design system; heavy runtime |
| Chakra | Runtime CSS-in-JS cost on data-dense pages |
| Bare CSS modules | Would require building an accessible component library from scratch |

**Why this combination:** Radix primitives give correct accessibility behaviour for the complex widgets this product needs — combobox, dialog, drawer, tooltip, popover, tabs — while Tailwind keeps styling co-located and produces no runtime cost. Components are copied into the codebase rather than installed as a dependency, so they can be adapted without fighting a library.

### Data tables — **TanStack Table with virtualisation**

The competitor listing table must handle 3,000+ rows with 24 configurable columns, sorting, filtering and export, and stay responsive. This is a hard requirement and it eliminates most alternatives. TanStack Table is headless — it provides the logic and leaves rendering to us, which is what allows virtualisation and custom cell rendering.

### Charts — **Recharts**

Sufficient for the chart types required (distributions, scatter, time series, radar, bubble), composable with React, and light enough to lazy-load. A lower-level library like D3 would give more control at a cost in development time that is not justified by the chart complexity here.

**Requirement on all charts:** every chart must have an accessible data-table alternative, and colour must never be the sole carrier of meaning.

### Server state — **TanStack Query**

Handles caching, background refetch, request deduplication and targeted invalidation. The key discipline: **a single map declares which queries each mutation invalidates**, so cache correctness is reviewable in one file rather than scattered across dozens of handlers.

### Client state — **Zustand, minimally**

Used only for wizard progress, multi-select sets and drawer state. Everything else is either server state or URL state. Filters, sort, tab and pagination live in the URL so that every view is shareable and survives a refresh.

**A deliberate constraint:** if a piece of state is being put into a client store, that is a signal to first ask whether it belongs in the URL or on the server.

---

## B.2 Backend

### Runtime and language — **Node.js LTS with TypeScript**

**The decisive argument is shared code.** Scoring formulas, statistical calculations, pricing arithmetic, Etsy constraint validators and score-band mappings are needed in *both* the browser (live validation, instant recalculation as the user moves a price slider) and the worker (batch computation). Writing them once, testing them once, and importing them in both places eliminates the single most dangerous class of bug in this product: **the frontend and backend disagreeing about a number the user is making financial decisions from.**

| Alternative | Assessment |
|---|---|
| **Python** | Better statistical and image libraries. But it would split the codebase in two, duplicate every validator and scoring formula, and require a second deployment pipeline. The statistical work here — proportions tests, rank correlations, effect sizes, ridge regression — is well within reach in TypeScript. |
| **Go** | Excellent for the worker, poor fit for sharing logic with the frontend, and slower to write the large amount of integration and orchestration code this product needs. |
| **Rust** | Wrong tool. This is an I/O-bound orchestration product, not a compute kernel. |

**Where Python may still appear:** if the learning loop's modelling outgrows what is comfortable in TypeScript, it can be isolated as a single scheduled job with a clean input/output contract, without affecting the rest of the system. This is a deliberate escape hatch, not a plan.

### API structure — **Typed RPC internally (tRPC), REST publicly (later)**

| Consideration | Conclusion |
|---|---|
| One consumer, one repository, one team | An interface definition language buys nothing today |
| ~60 distinct payload shapes | End-to-end type inference eliminates a large bug class |
| Future third-party access | A versioned REST façade over the same service layer, added when a consumer exists |

**The rule that makes this safe:** both surfaces are thin. All logic lives in the service layer. An RPC procedure and a future REST handler calling the same use case are each a handful of lines. This is what makes adding a public API a small project rather than a rewrite.

**Why not REST from the start:** it would mean hand-maintaining request/response types across a client and server that always deploy together — pure overhead with no consumer to justify it.

**Why not GraphQL:** the query patterns here are known and fixed. GraphQL's flexibility would buy nothing while adding N+1 risk, caching complexity and a resolver layer over data that is already shaped for its screens.

---

## B.3 Database

### Choice — **PostgreSQL**

| Requirement | PostgreSQL capability |
|---|---|
| Relational integrity across ~35 entities with deep lineage | Native, mature |
| High-volume immutable time series | Declarative range partitioning |
| Full-text search over titles, tags, descriptions | Generated search vectors, trigram matching |
| Vector similarity for concepts and images | Vector extension with approximate-nearest-neighbour indexing |
| Semi-structured provider payloads | Native JSON with indexing |
| Complex analytical aggregation | Window functions, CTEs, statistical aggregates |

| Alternative | Why not |
|---|---|
| MySQL | Weaker JSON, no comparable vector story, weaker analytical functions |
| MongoDB | The data is deeply relational. Lineage from listing → product → artwork → concept → gap → run is exactly what relational databases are for. |
| Postgres + Elasticsearch | Search here is navigational over tens of thousands of rows. A sync boundary and second system for that is unjustified. |
| Postgres + dedicated vector database | Similarity queries must combine with relational filters. A separate store makes that awkward and introduces embedding/row consistency problems. |

**Decision:** one database until measurements say otherwise. Documented exit criteria: vector index memory exceeding available RAM after partitioning, or analytical load measurably degrading transactional latency despite replicas.

### ORM — **Prisma, behind a repository layer**

| Alternative | Assessment |
|---|---|
| **Drizzle** | Closer to SQL, lighter runtime, better raw-SQL ergonomics. A genuinely close call. |
| **Kysely** | Excellent type-safe query building, but no migration story |
| **Raw SQL** | Maximum control, unacceptable maintenance cost at this schema size |

**Why Prisma:** the schema has roughly 35 tables with dense relationships. Prisma's generated types, relation handling and migration tooling materially reduce the volume of code and the opportunity for error. Its introspection and studio tooling also make the seeded development environment far more productive.

**The critical mitigation — a repository layer.** Application code never calls the ORM directly. It calls repositories. This means:

- Workspace scoping is applied in one place per aggregate, not at every call site.
- Read/write routing is centralised.
- Query instrumentation is universal.
- The three things the ORM handles awkwardly — partition management, vector similarity, concurrent index creation — are written as reviewed raw SQL inside repositories, without leaking elsewhere.
- If the ORM ever needs replacing, the blast radius is one directory.

---

## B.4 Storage

### Choice — **S3-compatible object storage (Cloudflare R2 as the default provider)**

| Requirement | Fit |
|---|---|
| Large binary assets | Native |
| Content-addressed keys | Application-controlled |
| Signed time-limited access | Native |
| Lifecycle rules for retention tiers | Native |
| Versioning for accidental deletion | Native |
| Low egress cost | **R2's decisive advantage** — this product serves many images repeatedly to the browser |

**Why R2 specifically:** artwork, mockups and competitor thumbnails are viewed repeatedly. Egress-free object storage removes a cost that would otherwise scale directly with usage. Because access is through the standard S3 API, the provider is interchangeable with S3, MinIO or any compatible service — no lock-in.

**Storage is never the database.** Metadata (dimensions, format, hash, purpose, lineage) lives in PostgreSQL; bytes live in object storage. This keeps database backups small and fast, which directly determines recovery time.

---

## B.5 AI

### Language and reasoning — **Claude, accessed through a tiered abstraction**

| Requirement | Why Claude fits |
|---|---|
| Reliable structured output | Strong instruction-following on schema-constrained generation, which the entire architecture depends on |
| Long context for aggregated statistics | Factor tables, gap tables and style summaries in a single grounded prompt |
| Multimodal analysis | Competitor thumbnail classification and artwork safety review use the same provider and the same contract |
| Multiple capability tiers | Opus-class for creative and reasoning work, Sonnet-class for analysis, Haiku-class for high-volume extraction — meaningful cost control without a second vendor |
| Prompt caching | Long shared context reused across sibling calls in one run |

**Architectural rule:** application code references **capability tiers**, never model names. Tier-to-model binding lives in configuration. A model upgrade is therefore a configuration change plus an evaluation run — not a code change. Exact model identifiers are pinned in environment configuration, never hard-coded.

### Image generation — **Ideogram, behind a provider interface**

| Requirement | Why Ideogram |
|---|---|
| **Legible typography inside generated images** | The decisive factor. POD artwork is disproportionately typography-led — slogan tees, badge lockups, arched text, pun mugs. Text rendering quality is where most image models fail and where this one is strongest. |
| Style control | Presets aligned to the six required design styles |
| Negative prompting | Needed to enforce the standing print constraints |
| Deterministic seeds | Required for reproducibility and for "same seed, adjusted brief" iteration |

**Provider-side prompt rewriting is disabled.** It would break reproducibility, defeat the negative constraints and make evaluation results meaningless.

**Portability is real, not theoretical.** Ideogram handles generation. Background removal, cropping, upscaling, print-readiness validation, rendition production, originality checking, vectorisation and safety review are all provider-independent — roughly 80% of the artwork pipeline. Swapping providers means a new adapter, six style-template mappings and an evaluation run.

### Embeddings — **A dedicated embedding model for text, an image-embedding model for visuals**

Used for concept deduplication, originality scoring and visual similarity. Stored in the database's vector columns so that similarity queries combine directly with relational filters.

---

## B.6 Automation

### Browser automation — **Not part of the core architecture**

This deserves a direct answer because it is the obvious thing to reach for when a data source has no API.

**Position:** browser automation is available as an **optional, off-by-default, operator-consented adapter** for retrieving data the operator is already entitled to see under their own subscription. It is **never** the default path, and the product must be fully functional without it.

**Hard constraints:**

| Constraint | Rule |
|---|---|
| Default state | Disabled. Requires explicit opt-in with a plain-language consent screen. |
| Credentials | The operator's own only. Never a shared or system account. |
| Rate | Roughly one request per three seconds, low daily ceiling, randomised jitter, no parallelism |
| Response to blocking | Halt immediately on a refusal or rate-limit response; open a circuit breaker for 24 hours |
| **Detection evasion** | **Never implemented.** No fingerprint spoofing, no CAPTCHA solving, no proxy rotation. If access requires evasion, the adapter reports itself unavailable. |
| Kill switch | One toggle disables it; the system falls back without interruption |

**This is enforced architecturally**, not by policy alone: the adapter interface has no capability for it, and a build-time check verifies no such logic exists in the codebase.

**The guaranteed path is operator-supplied file import** — the operator exports data from the tool they already subscribe to and uploads it. This path involves no automated access to anyone's systems and therefore cannot break.

### Job queue — **BullMQ on Redis**

| Alternative | Assessment |
|---|---|
| **Database-backed queue** | Fewer moving parts, but polling overhead and weaker primitives for concurrency limits, priorities and delayed jobs |
| **Cloud-managed queue (SQS)** | Vendor coupling, higher latency, weaker local development experience |
| **Temporal** | The right answer at much larger scale. Today it is a second control plane and a new failure mode for a single-user product. The step graph ports directly when the time comes. |

**Why BullMQ:** mature, well-instrumented, supports the specific features this product needs — per-queue concurrency limits, priorities, delayed and repeatable jobs, exponential backoff, dead-letter handling — and runs on Redis, which is needed anyway for caching, distributed rate limiting and progress fan-out.

**Critical clarification:** BullMQ carries *job delivery*, not *run state*. Run state lives in PostgreSQL. Redis is treated as ephemeral — if it were lost entirely, every incomplete run could be reconstructed and re-enqueued from the database. This separation is what makes the system safe to operate on a managed Redis with no durability guarantees.

---

## B.7 Hosting

| Tier | Recommendation | Reasoning |
|---|---|---|
| **Frontend / web** | Vercel | Purpose-built for this framework; atomic deploys with instant rollback; global edge caching for assets; preview environments per change |
| **Worker / scheduler** | Fly.io (or Railway) | Long-running processes with generous memory for image processing. Serverless functions are the wrong shape for a 12-minute pipeline step. |
| **Database** | Neon (or Supabase) | Managed PostgreSQL with point-in-time recovery and, in Neon's case, database branching — which gives production-shaped preview environments cheaply |
| **Cache / queue** | Upstash Redis | Managed, TLS, usage-based pricing appropriate to single-user volumes |
| **Object storage** | Cloudflare R2 | Egress-free, S3-compatible |
| **Observability** | Grafana Cloud + Sentry | Free tiers sufficient at Phase 1 volumes; standard protocols so the backend is replaceable |

**The portability principle:** no hosting-vendor-proprietary runtime API appears in application code. Everything runs as a plain Node process and a container. This topology is a preference informed by cost and convenience — not a dependency. Migrating any tier is an infrastructure change, not an application change.

---

## B.8 Summary of the stack

| Layer | Choice |
|---|---|
| Frontend framework | Next.js App Router, React, TypeScript |
| Styling / components | Tailwind CSS + Radix primitives |
| Tables / charts | TanStack Table (virtualised), Recharts |
| Server state / client state | TanStack Query / Zustand + URL state |
| Backend runtime | Node.js LTS, TypeScript strict |
| Internal API | Typed RPC (tRPC) |
| Public API | REST with an OpenAPI specification — later phase |
| Database | PostgreSQL with vector, trigram and partitioning extensions |
| Data access | Prisma behind a repository layer |
| Queue / cache / locks | Redis with BullMQ |
| Object storage | S3-compatible (R2 default) |
| AI — language and vision | Claude, tiered by capability |
| AI — images | Ideogram behind a provider interface |
| Image processing | Sharp plus segmentation, upscaling and vectorisation modules in the worker |
| Auth | Session-based with mandatory time-based one-time password |
| Hosting | Vercel (web) · Fly.io (worker) · Neon (database) · Upstash (Redis) · R2 (storage) |
| Infrastructure as code | Terraform, with Docker Compose for local development |
| Observability | OpenTelemetry → Grafana Cloud; Sentry for errors |

---

# Part C — Database Design

## C.1 Design principles

Seven rules govern every table. They are stated first because they explain most of the structure that follows.

### 1. Every table carries a workspace identifier — from the first migration

Even with one user. The column costs nothing now; adding it to 35 tables holding terabytes of data later is a multi-month project with genuine corruption risk. Every composite index leads with it, so queries are efficient the day multi-user arrives.

### 2. Identity is separated from observation

A competitor shop *exists* — that is a stable fact. Its estimated monthly revenue *was observed on a date* — that is a volatile fact. These live in different tables.

```
COMPETITOR SHOP (identity)     ──┐
  name, marketplace id,          │  stable, updated rarely
  opened date                    │
                                 ├──►  observations accumulate
COMPETITOR LISTING (identity)  ──┤     over time, never overwrite
  title, marketplace id,         │
  product type, first seen       │
                                 │
LISTING SNAPSHOT (observation) ──┘  immutable, dated, partitioned
  price, sales estimate, reviews,
  image count, captured at
```

**This single decision enables:** longitudinal comparison of the same market over time, re-analysis without re-fetching, honest provenance for every figure, and correction without destruction.

### 3. Snapshots are immutable

Observation tables have no update timestamp because they are never updated. A correction is a new row. This is what makes any historical report reproducible.

### 4. Every derived number records the configuration that produced it

Scores are computed from weights. Weights change as the system learns. Therefore every score stores the version of the scoring configuration that produced it — otherwise historical scores become uninterpretable the moment weights are updated.

### 5. Money is integers plus a currency code

Minor units (pence, cents) as whole numbers, with an explicit currency column. Never floating point. Rounding errors in margin calculations would directly cause the user to make bad pricing decisions.

### 6. Estimated data is marked as estimated

Every value that came from a third-party estimate carries its source and a confidence level. The interface is required to display these differently, and it can only do so if the database records them.

### 7. High-volume tables are partitioned from day one

Five tables will dominate row count. All are declared as time-partitioned from the first migration, even with a single partition present.

---

## C.2 Schema map

```
IDENTITY & CONFIGURATION
   users · workspaces · workspace members · sessions
   settings · scoring configurations · fee models · product types

RESEARCH
   niches · sub-niches · research runs · run steps · run events

MARKET DATA
   competitor shops · run shop selections · competitor listings
   listing snapshots · listing analysis · keyword statistics
   market data imports

ANALYSIS
   opportunity reports · success reports · success attributes
   failure reports · failure attributes · coverage cells · market gaps

CREATIVE
   design concepts · concept scores · legal screenings · legal matches
   artwork briefs · generated artwork · artwork renditions · artwork QA

COMMERCE
   blueprints · print providers · blueprint variants
   product recommendations · product drafts · product variants
   SEO variations · pricing snapshots

PUBLISHING
   integrations · integration credentials · printify products
   etsy listings · listing images · publish jobs

ANALYTICS & LEARNING
   performance snapshots · outcome records · prediction records
   model proposals

PLATFORM
   files · ai prompt calls · provider calls · audit log
   notifications · research history (view)
```

---

## C.3 Identity and configuration

### `users`

The person operating the system. One row in Phase 1.

| Field | Type | Notes |
|---|---|---|
| **id** (PK) | UUID | Time-ordered for index locality |
| email | text, unique | Login identifier |
| password_hash | text | Memory-hard hash with versioned parameters |
| totp_secret_encrypted | binary | Envelope-encrypted second-factor secret |
| totp_enabled | boolean | Mandatory in practice |
| recovery_codes_encrypted | binary | Single-use, hashed |
| display_name | text | |
| last_login_at | timestamp | |
| failed_login_count | integer | Drives lockout |
| locked_until | timestamp | Exponential lockout |
| created_at / updated_at / deleted_at | timestamp | Soft delete |

**Relationships:** one user has many workspace memberships, many sessions.

---

### `workspaces`

**The tenancy boundary.** Every other table in the system is scoped to a workspace. One row in Phase 1.

| Field | Type | Notes |
|---|---|---|
| **id** (PK) | UUID | |
| name | text | |
| slug | text, unique | Future URL scoping |
| base_currency | char(3) | Default for all money display |
| country_code | char(2) | Drives tax treatment |
| timezone | text | Report boundaries, scheduling |
| plan | text | `personal` now; subscription tier later |
| status | text | `active` / `suspended` / `closed` |
| created_at / updated_at / deleted_at | timestamp | |

**Reasoning.** This table exists in Phase 1 with a single row that is never displayed to the user. Its entire purpose is to make the multi-user conversion additive rather than structural.

---

### `workspace_members`

Links users to workspaces with a role. One row in Phase 1, role `owner`.

| Field | Type | Notes |
|---|---|---|
| **id** (PK) | UUID | |
| workspace_id (FK) | UUID → workspaces | |
| user_id (FK) | UUID → users | |
| role | enum | `owner` / `admin` / `member` / `viewer` |
| invited_by (FK) | UUID → users | Null for the founding owner |
| joined_at | timestamp | |

**Constraint:** unique on (workspace_id, user_id).

**Reasoning.** The role enum and permission matrix are defined now and enforced from the multi-user phase. Defining them later would mean auditing every endpoint retrospectively.

---

### `sessions`

| Field | Type | Notes |
|---|---|---|
| **id** (PK) | UUID | |
| user_id (FK) | UUID → users | |
| workspace_id (FK) | UUID → workspaces | Active workspace for this session |
| token_hash | text, unique | Opaque token hashed at rest |
| ip_hash | text | Hashed, not raw — privacy |
| user_agent | text | |
| expires_at | timestamp | Absolute expiry |
| revoked_at | timestamp | Instant revocation |
| created_at | timestamp | |

---

### `settings`

Workspace-level configuration that the operator controls.

| Field | Type | Notes |
|---|---|---|
| **id** (PK) | UUID | |
| workspace_id (FK) | UUID → workspaces, unique | One row per workspace |
| default_run_depth | enum | `quick` / `standard` / `deep` |
| per_run_budget_amount | bigint | Minor units |
| monthly_budget_amount | bigint | |
| budget_currency | char(3) | |
| behaviour_at_cap | enum | `pause` / `block` |
| margin_floor_percent | decimal | Hard constraint on pricing |
| default_artwork_variants | integer | Typically 4 |
| market_data_adapter | text | Chosen acquisition method |
| amber_adapters_enabled | boolean | Off by default |
| ai_disclosure_in_listings | boolean | User's choice |
| notification_preferences | JSON | Per-event channel matrix |
| retention_overrides | JSON | |
| updated_at | timestamp | |

**Reasoning for a single settings row rather than key-value pairs.** These settings are read on nearly every operation and are strongly typed with real constraints (a margin floor is a decimal between 0 and 1). A key-value table would trade type safety and a single fast read for flexibility that is not needed.

---

### `scoring_configurations`

**One of the most important tables in the system.** Every score in the product is produced by a versioned set of weights, and this is where they live.

| Field | Type | Notes |
|---|---|---|
| **id** (PK) | UUID | |
| workspace_id (FK) | UUID → workspaces | |
| version | integer | Sequential per workspace |
| name | text | Human label |
| source | enum | `expert_prior` / `fitted` / `manual` |
| is_active | boolean | At most one active per workspace |
| weights | JSON | Full weight tree for every score |
| normalisation | JSON | Per-feature scaling parameters |
| thresholds | JSON | Score bands, minimum sample sizes, demand floor |
| fitted_from | JSON | Outcome count, date range, method, accuracy metrics |
| activated_at / activated_by | timestamp / UUID | |
| notes | text | Why this version exists |
| created_at | timestamp | |

**Constraints:** unique on (workspace_id, version); a partial unique index enforces at most one active configuration per workspace.

**Reasoning — why weights are data, not code.** Three requirements force this:
1. The learning loop must be able to *propose* new weights, which means weights must be storable and comparable.
2. Historical scores must remain interpretable, which means each score must point at the weights that produced it.
3. A bad configuration must be revertible in one action, without a deployment.

Storing weights as JSON rather than as columns is deliberate: the weight tree is nested and evolves as new scores are added, and it is always read whole, never queried by individual weight.

---

### `fee_models`

The marketplace fee structure used for every profit calculation.

| Field | Type | Notes |
|---|---|---|
| **id** (PK) | UUID | |
| workspace_id (FK) | UUID → workspaces | |
| name | text | |
| listing_fee_amount | bigint | Fixed per listing |
| transaction_percent | decimal | |
| payment_percent | decimal | |
| payment_fixed_amount | bigint | |
| offsite_ads_percent | decimal | |
| offsite_ads_applies | boolean | **Defaults true** |
| regulatory_percent | decimal | |
| vat_percent | decimal | |
| vat_applies | boolean | |
| is_default | boolean | |
| created_at / updated_at | timestamp | |

**Reasoning.** Fees change. When they do, every unpublished product's margin must be recalculated and every historical calculation must remain explicable. Making the fee model a versioned row rather than a constant achieves both. The `offsite_ads_applies` default of true encodes the product principle that margin must be modelled pessimistically.

---

### `product_types`

Reference data. The six supported physical product categories.

| Field | Type | Notes |
|---|---|---|
| **id** (PK) | UUID | |
| code | text, unique | `tshirt`, `sweatshirt`, `hoodie`, `mug`, `poster`, `tote` |
| name | text | Display label |
| category | text | `apparel` / `drinkware` / `wall_art` / `accessory` |
| default_print_area | JSON | Width, height, required resolution |
| marketplace_taxonomy_id | integer | Default listing category |
| is_active | boolean | |

**Note:** this is global reference data, not workspace-scoped — one of only three tables in the system that is not.

---

## C.4 Research

### `niches`

A market the operator has investigated. Persistent across runs.

| Field | Type | Notes |
|---|---|---|
| **id** (PK) | UUID | |
| workspace_id (FK) | UUID → workspaces | |
| name | text | As entered |
| normalised_name | text | Lowercased, trimmed, collapsed — used for matching |
| description | text | |
| status | enum | `active` / `rejected` / `exhausted` |
| rejection_reason | text | Why the operator declined it |
| best_opportunity_score | decimal | Highest achieved across runs |
| last_researched_at | timestamp | |
| created_at / updated_at / deleted_at | timestamp | |

**Constraints:** unique on (workspace_id, normalised_name). A trigram index supports fuzzy matching so that "gardening" and "Gardening " resolve to the same niche.

**Reasoning for the rejection fields.** A niche the operator investigated and declined is *valuable information*. Without recording it, they re-research the same dead market six months later having forgotten. Rejection is a first-class outcome, not an absence.

---

### `sub_niches`

Discovered segments within a niche.

| Field | Type | Notes |
|---|---|---|
| **id** (PK) | UUID | |
| workspace_id (FK) | UUID → workspaces | |
| niche_id (FK) | UUID → niches | |
| run_id (FK) | UUID → research_runs | Which run discovered it |
| name / normalised_name | text | |
| description | text | |
| rationale | text | Why this is a real segment |
| demand_index | decimal | |
| competition_index | decimal | |
| opportunity_score | decimal | Its own independent score |
| rank | integer | Within the run |
| example_search_terms | text array | |
| evidence_sources | text array | Which signals supported it |
| listings_observed | integer | Supply count |
| scoring_config_version | integer | |
| created_at | timestamp | |

**Reasoning for `evidence_sources`.** Sub-niches are discovered from three independent signals — domain reasoning, co-occurrence in observed listing text, and marketplace taxonomy. A sub-niche supported by all three is far more trustworthy than one supported by reasoning alone, and the interface must be able to show this.

---

### `research_runs`

**The central work unit.** One execution of the pipeline. The unit of cost, audit and resumption.

| Field | Type | Notes |
|---|---|---|
| **id** (PK) | UUID | |
| workspace_id (FK) | UUID → workspaces | |
| niche_id (FK) | UUID → niches | |
| product_type_id (FK) | UUID → product_types | |
| requested_style | enum | Including `auto` |
| resolved_style | enum | What the data chose |
| resolved_style_reason | text | Evidence for the choice |
| depth | enum | `quick` / `standard` / `deep` |
| status | enum | See state list below |
| seed_keywords / excluded_terms | text array | |
| is_degraded | boolean | Ran with reduced capability |
| degradation_reasons | text array | Exactly what was missing |
| scoring_config_id (FK) | UUID → scoring_configurations | |
| budget_amount / spend_amount | bigint | Minor units |
| budget_currency | char(3) | |
| idempotency_key | text | Prevents accidental duplicate runs |
| parent_run_id (FK) | UUID → research_runs | Clone and refine lineage |
| started_at / finished_at / cancelled_at | timestamp | |
| error | JSON | |
| created_by (FK) | UUID → users | |
| created_at / updated_at / deleted_at | timestamp | |

**Run statuses:** `queued`, `running`, `paused_budget`, `awaiting_selection`, `partially_failed`, `failed`, `completed`, `cancelled`.

**Reasoning for `awaiting_selection` as a status.** The human concept-selection gate could have been modelled as a paused job. Making it a first-class run status instead means the run can wait indefinitely without holding any queue resource, survives any restart, and is queryable ("show me everything waiting for my decision"). The same mechanism supports future gates without new machinery.

---

### `run_steps`

**The backbone of the entire system.** One row per pipeline step. This is what makes runs durable and resumable.

| Field | Type | Notes |
|---|---|---|
| **id** (PK) | UUID | |
| workspace_id | UUID | Denormalised for index efficiency |
| run_id (FK) | UUID → research_runs | |
| step_key | text | `opportunity`, `competitors`, `success`, … |
| step_order | integer | |
| status | enum | `pending` / `running` / `succeeded` / `failed` / `skipped` / `blocked_external` / `cancelled` |
| attempt / max_attempts | integer | |
| depends_on | text array | Step keys that must succeed first |
| input_hash | text | Detects whether re-execution is needed |
| output_reference | JSON | Pointer `{table, id}` to what was produced |
| cost_amount / cost_currency | bigint / char(3) | |
| tokens_in / tokens_out | bigint | AI usage |
| started_at / finished_at / duration_ms | timestamp / integer | |
| error | JSON | |
| created_at / updated_at | timestamp | Heartbeat uses updated_at |

**Constraints:** unique on (run_id, step_key).

**Reasoning — three decisions worth explaining.**

*Why `output_reference` is a JSON pointer rather than fifteen nullable foreign keys.* Each step produces a different entity type. Fifteen mostly-null columns would be unusable; a `{table, id}` pointer keeps the table generic while remaining fully traceable.

*Why `depends_on` is stored per row rather than only in code.* The step graph is declarative data. Storing dependencies on the row means the orchestrator can determine what is unblocked with a single query, and the graph is inspectable in the database when diagnosing a stuck run.

*Why `updated_at` doubles as a heartbeat.* A step running for longer than its expected duration with no recent update means its worker died. A maintenance task finds these and requeues them. This is the entire crash-recovery mechanism, and it costs one column.

---

### `run_events`

Progress event stream. Partitioned monthly, retained 90 days.

| Field | Type | Notes |
|---|---|---|
| **id** (PK, composite with timestamp) | UUID | |
| workspace_id / run_id | UUID | |
| step_key | text | |
| level | text | `info` / `warn` / `error` |
| event | text | Machine-readable event type |
| message | text | Human-readable progress text |
| data | JSON | |
| created_at (partition key) | timestamp | |

**Reasoning.** These are what the live progress display consumes, and what allows a user reconnecting mid-run to see everything they missed. They are high-volume and low-value after the run completes — hence partitioning and short retention.

---

### `research_history` (view, not a table)

The operator-facing history is a **view** over runs joined to niches, product types and report summaries.

**Reasoning.** History is a presentation of existing data. Materialising it as its own table would create a synchronisation problem for no benefit. If it becomes slow at scale, it is promoted to a materialised view refreshed on run completion — a change invisible to the application.

---

## C.5 Market data

### `competitor_shops`

Stable identity and slow-moving facts about an observed shop.

| Field | Type | Notes |
|---|---|---|
| **id** (PK) | UUID | |
| workspace_id (FK) | UUID → workspaces | |
| marketplace_shop_id | bigint | Etsy's identifier |
| shop_name | text | |
| shop_url | text | |
| opened_at | date | |
| age_months | integer | Derived, refreshed on observation |
| location | text | |
| total_sales | integer | |
| total_reviews | integer | |
| average_rating | decimal | |
| active_listing_count | integer | |
| estimated_monthly_sales | integer | |
| estimated_monthly_revenue_amount | bigint | |
| revenue_currency | char(3) | |
| review_velocity_90d | decimal | Reviews per month, trailing 90 days |
| data_source | text | Which adapter supplied this |
| data_confidence | enum | `low` / `medium` / `high` |
| first_seen_at / last_seen_at | timestamp | |
| created_at / updated_at | timestamp | |

**Constraints:** unique on (workspace_id, marketplace_shop_id).

**Reasoning for shops being workspace-scoped rather than global.** Two workspaces observing the same shop may have different data sources with different confidence. Sharing the row would mean one workspace's lower-quality data degrading another's. Shared *market facts* (such as visual analysis keyed by image content) are shared at the platform level later; shop records are not.

---

### `run_shop_selections`

The join recording **why** a shop was chosen for a particular run.

| Field | Type | Notes |
|---|---|---|
| **id** (PK) | UUID | |
| workspace_id | UUID | |
| run_id (FK) | UUID → research_runs | |
| shop_id (FK) | UUID → competitor_shops | |
| selection_score | decimal | |
| rank | integer | |
| criteria_passed | JSON | Which gates it cleared, what age multiplier applied |
| listings_collected | integer | |
| was_truncated | boolean | Hit the per-shop cap |
| created_at | timestamp | |

**Reasoning.** Shops are reused across runs, so this must be a join table rather than a foreign key on the shop. More importantly, `criteria_passed` is what allows the interface to answer "why is this shop in my analysis?" — which is a requirement, not a nicety.

---

### `competitor_listings`

Stable identity of an observed listing.

| Field | Type | Notes |
|---|---|---|
| **id** (PK) | UUID | |
| workspace_id (FK) | UUID → workspaces | |
| shop_id (FK) | UUID → competitor_shops | |
| marketplace_listing_id | bigint | |
| title | text | |
| url | text | |
| product_type_id (FK) | UUID → product_types | |
| listed_at | date | Determines listing age |
| first_seen_at / last_seen_at | timestamp | |
| is_active | boolean | |
| title_search_vector | generated | Full-text index |
| created_at / updated_at | timestamp | |

**Constraints:** unique on (workspace_id, marketplace_listing_id).

---

### `listing_snapshots`

**Immutable observations.** Partitioned monthly. The highest-volume table in the system.

| Field | Type | Notes |
|---|---|---|
| **id** (PK, composite with captured_at) | UUID | |
| workspace_id / run_id / listing_id | UUID | |
| captured_at (partition key) | timestamp | |
| price_amount / price_currency | bigint / char(3) | |
| shipping_amount | bigint | |
| free_shipping | boolean | |
| image_count | integer | |
| primary_image_file_id (FK) | UUID → files | |
| description | text | |
| tags | text array | |
| materials | text array | |
| review_count | integer | |
| average_rating | decimal | |
| review_velocity_90d | decimal | |
| favourites | integer | |
| views | integer | Rarely available for competitors |
| estimated_monthly_sales | integer | |
| estimated_total_sales | integer | |
| estimated_monthly_revenue_amount | bigint | |
| is_bestseller | boolean | |
| has_personalisation | boolean | |
| listing_age_days | integer | At capture time |
| shop_age_months | integer | At capture time |
| section | text | |
| data_source | text | |
| data_confidence | enum | |
| raw_payload | JSON | Original provider response |

**No update timestamp. This table is never updated.**

**Reasoning — why this design is load-bearing.**

*Longitudinal analysis becomes possible.* Review velocity is computed by comparing snapshots. Trend direction is computed from the slope of repeated observations. Neither is possible if each observation overwrites the last.

*Re-analysis without re-fetching.* When the visual analysis model improves, every stored snapshot can be re-analysed without touching the marketplace or paying for the data again.

*Age is captured at observation time, not derived at read time.* A listing's age when it was observed is a fact about that observation. Computing it later from the current date would silently corrupt every historical cohort calculation.

*Provenance travels with the data.* `data_source` and `data_confidence` on the row mean any figure in any report can be traced to where it came from.

---

### `listing_analysis`

The extracted visual and content characteristics of a listing image. **This table is the product's core differentiator expressed as data.**

| Field | Type | Notes |
|---|---|---|
| **id** (PK) | UUID | |
| workspace_id | UUID | |
| listing_id (FK) | UUID → competitor_listings | |
| snapshot_id | UUID | Which observation this analysed |
| image_hash | text | SHA-256 of the analysed image |
| palette | JSON | Extracted colours with proportions and perceptual values |
| palette_family | enum | `muted_earth`, `muted_green`, `high_contrast_mono`, `pastel`, `neon`, `vintage_washed`, `jewel`, `monochrome_dark`, `natural_neutral`, `warm_retro`, `cool_modern`, `other` |
| dominant_colour | text | |
| typography_style | enum | `vintage_serif`, `condensed_sans`, `script`, `handwritten`, `slab`, `display_bold`, `retro_groovy`, `minimal_sans`, `distressed`, `none` |
| typography_confidence | decimal | |
| layout_archetype | enum | `centred_stack`, `badge_circle`, `arched_text`, `left_aligned_block`, `illustration_only`, `text_over_illustration`, `split`, `border_frame`, `repeat_pattern`, `other` |
| mockup_style | enum | `flat_lay`, `model_lifestyle`, `ghost_mannequin`, `hanging`, `folded`, `studio_plain`, `graphic_only`, `in_situ_scene`, `other` |
| subject_tags | text array | |
| humour_type | text | `pun` / `sarcasm` / `wholesome` / `none` |
| has_text | boolean | |
| text_length_band | text | `none` / `short` / `medium` / `long` |
| garment_colour | text | |
| visual_embedding | vector | For similarity and originality |
| analyser_version | text | Which model version produced this |
| analysed_at | timestamp | |

**Constraints:** unique on (workspace_id, image_hash, analyser_version).

**Reasoning — the uniqueness constraint is a major cost decision.** Keying on image content hash rather than on listing identity means:
- The same image appearing in two listings is analysed once.
- Re-running a niche weeks later reuses all prior analysis.
- After the third run in a niche, cache hit rates typically exceed 40%.

Vision analysis is the largest per-run AI cost in the system. This constraint roughly halves it on repeat research.

**Why controlled vocabularies rather than free text.** Every one of these fields is used for statistical aggregation — counting how many top performers use vintage serif typography. Free-text classification would make aggregation meaningless. The enums also let the system reject out-of-vocabulary model output rather than silently accepting an invented category.

---

### `keyword_statistics`

Term frequency and performance analysis across a run's collected listings.

| Field | Type | Notes |
|---|---|---|
| **id** (PK) | UUID | |
| workspace_id / run_id | UUID | |
| term | text | |
| term_type | enum | `tag` / `title_phrase` / `description_phrase` |
| frequency | integer | Total occurrences |
| document_frequency | integer | Listings containing it |
| relevance_score | decimal | Weighted against a general baseline |
| sales_weighted_score | decimal | **The important one** — weighted by the sales of listings using it |
| top_decile_usage_percent | decimal | |
| average_price_amount | bigint | |
| competition_index | decimal | |
| created_at | timestamp | |

**Constraints:** unique on (run_id, term, term_type).

**Reasoning for `sales_weighted_score`.** Raw frequency identifies common words, not effective ones. A term used by 200 listings that sell nothing is worthless; a term used by 12 listings that all sell well is gold. Weighting by the performance of the listings that use a term is what makes the SEO engine evidence-based rather than a frequency counter.

---

### `market_data_imports`

Record of operator-supplied data files.

| Field | Type | Notes |
|---|---|---|
| **id** (PK) | UUID | |
| workspace_id | UUID | |
| source | text | Which tool the export came from |
| filename | text | |
| file_id (FK) | UUID → files | |
| file_hash | text | Prevents duplicate import |
| row_count / accepted_count / rejected_count | integer | |
| column_mapping | JSON | Remembered per file signature |
| errors | JSON | Per-row rejection reasons |
| imported_by (FK) | UUID → users | |
| created_at | timestamp | |

**Reasoning.** The operator-file path is the guaranteed market data route. Recording the mapping means the operator confirms column mapping once per file format, not once per import. Recording the hash means an accidental re-upload is detected rather than duplicating thousands of observations.

---

## C.6 Analysis

### `opportunity_reports`

The niche verdict. Versioned and immutable.

| Field | Type | Notes |
|---|---|---|
| **id** (PK) | UUID | |
| workspace_id / run_id / niche_id | UUID | |
| version | integer | |
| is_provisional | boolean | Early estimate, later superseded |
| overall_score | decimal | 0–100 |
| verdict | enum | `avoid` / `marginal` / `good` / `strong` / `exceptional` |
| demand_score | decimal | |
| competition_score | decimal | |
| trend_score | decimal | |
| profitability_score | decimal | |
| seasonality_score | decimal | |
| sub_score_details | JSON | Per sub-score: input features, method, confidence, explanation |
| seasonality_index | decimal array(12) | Monthly demand index |
| peak_months / trough_months | integer array | |
| seasonality_strength | decimal | |
| executive_summary | text | |
| percentile_in_workspace | decimal | Rank against prior research |
| is_degraded | boolean | |
| scoring_config_id (FK) | UUID → scoring_configurations | |
| created_at | timestamp | |

**Constraints:** unique on (run_id, version).

**Reasoning for versioning and `is_provisional`.** The system emits an early provisional score for perceived speed, then a final score once competitor data lands. Both must coexist so the interface can show the transition honestly rather than silently changing a number the user has already read.

**Reasoning for `sub_score_details` as JSON.** Each sub-score has a different set of input features. Modelling this relationally would require a feature table with a type discriminator, for data that is always read as a whole with its parent and never queried independently.

---

### `success_reports` and `success_attributes`

**`success_reports`** — the analysis header.

| Field | Type | Notes |
|---|---|---|
| **id** (PK) | UUID | |
| workspace_id / run_id | UUID | |
| cohort_definition | JSON | Metric, percentile, minimum age, sample sizes |
| cohort_size / population_size | integer | |
| synthesis | JSON | The "median winning listing" specification |
| correlations | JSON | Rank correlation matrix, multicollinearity flags |
| interactions | JSON | Attribute pairs whose combined effect exceeds their individual effects |
| resolved_style | enum | |
| resolved_style_evidence | JSON | |
| scoring_config_id (FK) | UUID | |
| created_at | timestamp | |

**`success_attributes`** — the individual findings. *This is the table that produces the sentences the user reads.*

| Field | Type | Notes |
|---|---|---|
| **id** (PK) | UUID | |
| workspace_id | UUID | |
| success_report_id (FK) | UUID → success_reports | |
| attribute_group | text | `colour` / `typography` / `layout` / `pricing` / `presentation` / `seo` / `format` |
| attribute | text | e.g. `palette_family` |
| value | text | e.g. `muted_green` |
| numeric_band | JSON | For numeric attributes: the optimal range |
| statement | text | Rendered sentence |
| **cohort_support_percent** | decimal | Prevalence among top performers |
| **baseline_support_percent** | decimal | **Prevalence across the whole market** |
| **lift** | decimal | Ratio of the two |
| **sample_size** | integer | |
| p_value | decimal | Statistical significance |
| effect_size | decimal | |
| confidence | enum | `low` / `medium` / `high` |
| weight | decimal | Strength for downstream scoring |
| insufficient_evidence | boolean | Below thresholds — excluded from ranking |
| supporting_listing_ids | UUID array | Click-through evidence |
| rank | integer | |
| created_at | timestamp | |

**Reasoning — the four bolded fields are the product's integrity guarantee.**

Storing `cohort_support_percent` alone would allow the interface to display "84% of top performers use muted greens" — which is *actively misleading* if 82% of all listings use muted greens. By storing baseline and lift as required fields alongside it, the schema makes the honest presentation the natural one and the dishonest one impossible to produce accidentally.

`sample_size` and `insufficient_evidence` implement the other integrity requirement: findings from too little data are stored (so the user can see they were considered) but flagged so they never enter the ranked list or influence scoring.

`supporting_listing_ids` is what makes "show me the 35 listings" a single click. Storing the identifiers at analysis time is far cheaper than reconstructing the cohort query later.

---

### `failure_reports` and `failure_attributes`

Structurally parallel to the success tables, with two additional fields on the attribute table:

| Field | Type | Notes |
|---|---|---|
| penalty_weight | decimal | Strength as a negative signal |
| causality | enum | `causal_plausible` / `correlational_only` |
| is_ambiguous | boolean | Also appears as a success attribute |

**Reasoning for `causality`.** "Listings with one image underperform" has an obvious mechanism. "Listings using orange underperform" does not. Presenting both with equal authority would mislead. This field is set by explicit assessment and drives different presentation in the interface.

**Reasoning for `is_ambiguous`.** Some attributes appear in both reports — common among winners *and* among losers, meaning the attribute is simply common. Rather than silently suppressing these, the schema marks them so the net effect can be shown and their downstream weight reduced.

---

### `coverage_cells` and `market_gaps`

**`coverage_cells`** — the supply-versus-demand map.

| Field | Type | Notes |
|---|---|---|
| **id** (PK) | UUID | |
| workspace_id / run_id | UUID | |
| sub_niche_id (FK) | UUID → sub_niches | |
| design_angle | text | |
| style | enum | |
| supply_count | integer | Observed competing listings |
| demand_index | decimal | |
| monetisability | decimal | Price power in this segment |
| average_price_amount | bigint | |
| created_at | timestamp | |

**Constraints:** unique on (run_id, sub_niche_id, design_angle, style).

**`market_gaps`** — the cells that qualify as opportunities.

| Field | Type | Notes |
|---|---|---|
| **id** (PK) | UUID | |
| workspace_id / run_id | UUID | |
| coverage_cell_id (FK) | UUID → coverage_cells | |
| sub_niche_id (FK) | UUID → sub_niches | |
| title | text | |
| design_angle | text | |
| style | enum | |
| gap_score | decimal | 0–100 |
| demand_index | decimal | |
| supply_count | integer | |
| monetisability / feasibility | decimal | |
| explanation | text | |
| demand_evidence / supply_evidence | JSON | **Required** |
| suggested_angles | text array | |
| caution_flags | text array | `trademark_heavy` / `seasonal_dead` / `unprintable` |
| rank | integer | |
| scoring_config_id (FK) | UUID | |
| created_at | timestamp | |

**Reasoning for separating cells from gaps.** The coverage matrix contains every combination examined, including those rejected by the demand floor. Keeping both means the visualisation can show the full map with the floor drawn on it, and the user can verify for themselves that excluded cells were excluded for the right reason. Storing only the qualifying gaps would make that impossible.

**Reasoning for `demand_evidence` being required.** The single most likely way this engine embarrasses the product is by recommending an empty market that is empty because nobody wants it. Making evidence a required field on every returned gap forces the engine to justify itself and lets the interface display that justification.

---

## C.7 Creative

### `design_concepts`

| Field | Type | Notes |
|---|---|---|
| **id** (PK) | UUID | |
| workspace_id (FK) | UUID → workspaces | |
| run_id (FK) | UUID → research_runs | |
| niche_id / sub_niche_id (FK) | UUID | |
| market_gap_id (FK) | UUID → market_gaps | Null for success-derived concepts |
| parent_concept_id (FK) | UUID → design_concepts | For expansions |
| origin | enum | `success` / `gap` / `manual` / `expansion` |
| status | enum | `generated` / `selected` / `screening` / `cleared` / `blocked` / `artwork_pending` / `artwork_ready` / `archived` / `rejected` |
| name | text | |
| description | text | |
| target_audience | text | |
| design_angle | text | |
| style | enum | |
| visual_direction | JSON | Palette family, colours, typography class, layout archetype |
| text_content | text | Words appearing in the design, if any |
| reasoning | text | Why this should work |
| cited_success_attribute_ids | UUID array | **Traceable evidence** |
| cited_gap_ids | UUID array | |
| text_embedding | vector | Deduplication and originality |
| near_duplicate_of (FK) | UUID → design_concepts | |
| similarity_score | decimal | |
| prompt_version | text | Which prompt produced it |
| selected_at | timestamp | |
| created_at / updated_at / deleted_at | timestamp | |

**Reasoning for the citation arrays.** A concept's reasoning claims things like "muted earth palettes show 2.1× lift here". Storing the identifiers of the attributes actually cited means the interface can link straight to that finding, and an automated check can verify the claim matches the stored statistic. Without these, the reasoning would be unverifiable prose.

**Reasoning for the embedding.** Twenty concepts must be genuinely distinct. Similarity is computed at generation time and used to reject and regenerate near-duplicates. Storing it also allows checking against the workspace's entire concept history, preventing the operator from unknowingly making the same design twice.

---

### `concept_scores`

Append-only. One row per scoring event.

| Field | Type | Notes |
|---|---|---|
| **id** (PK) | UUID | |
| workspace_id | UUID | |
| concept_id (FK) | UUID → design_concepts | |
| artwork_id (FK) | UUID → generated_artwork | Null at concept stage |
| stage | enum | `concept` / `artwork` |
| market_fit | decimal | |
| originality | decimal | |
| conversion | decimal | |
| competition | decimal | |
| opportunity | decimal | |
| design_success_score | decimal | Combined |
| band | text | `weak` … `exceptional` |
| contributions | JSON | Signed point contribution per factor |
| reasoning | JSON | Rendered from contributions |
| feature_snapshot | JSON | Exact inputs, for later back-testing |
| scoring_config_id (FK) | UUID | |
| created_at | timestamp | |

**Reasoning for append-only with a stage discriminator.** A concept is scored twice — once as text, once after artwork exists, because originality and conversion signals change when the image is real. Keeping both allows measuring which stage predicts outcomes better. Overwriting would destroy that.

**Reasoning for `feature_snapshot`.** When the learning loop later evaluates prediction accuracy, it needs the exact inputs the original prediction used — not today's recomputed values, which would leak hindsight into the evaluation.

---

### `legal_screenings` and `legal_matches`

**`legal_screenings`**

| Field | Type | Notes |
|---|---|---|
| **id** (PK) | UUID | |
| workspace_id | UUID | |
| subject_type | enum | `concept` / `artwork` / `listing_text` |
| subject_id | UUID | Polymorphic reference |
| risk_level | enum | `none` / `low` / `medium` / `high` / `blocked` |
| extracted_entities | JSON | Brands, characters, slogans found |
| copyright_assessment | JSON | |
| marketplace_policy_flags | text array | |
| rationale | text | |
| safer_alternatives | JSON | |
| was_overridden | boolean | |
| overridden_by (FK) | UUID → users | |
| override_justification | text | |
| overridden_at | timestamp | |
| screener_version | text | |
| created_at | timestamp | |

**`legal_matches`** — individual registry or blocklist hits.

| Field | Type | Notes |
|---|---|---|
| **id** (PK) | UUID | |
| screening_id (FK) | UUID → legal_screenings | |
| matched_term | text | |
| match_type | enum | `exact` / `fuzzy` / `phonetic` / `blocklist` |
| registry | text | Which authority |
| registration_number | text | |
| owner_name | text | |
| goods_classes | integer array | |
| jurisdiction | text | |
| mark_status | text | |
| registry_url | text | |
| similarity | decimal | |
| raw_response | JSON | |
| created_at | timestamp | |

**Reasoning for polymorphic subject reference.** Concepts, artwork and listing text all require screening with identical semantics and identical output. Three parallel table sets would triple the surface area for a difference that does not exist. This is the one place polymorphism is accepted, and it is constrained to a fixed set of subject types.

**Reasoning for retaining raw registry responses for seven years.** If a dispute ever arises, the operator needs to demonstrate what was checked and what the authority returned at the time. This is the longest retention period in the system, and it is deliberate.

**Reasoning for override fields being on the screening rather than a separate table.** An override is an attribute of a specific screening decision, not an independent entity. Keeping them together means the override can never become detached from what it overrode.

---

### `artwork_briefs`

| Field | Type | Notes |
|---|---|---|
| **id** (PK) | UUID | |
| workspace_id | UUID | |
| concept_id (FK) | UUID → design_concepts | |
| version | integer | |
| subject | text | |
| composition | text | |
| palette_colours | text array | Explicit values, derived from the winning palette family |
| typography_direction | JSON | Class, weight, case, arrangement — never a font name |
| texture_finish | text | |
| background_requirement | text | Always transparent |
| aspect_ratio | text | |
| target_print_dimensions | JSON | Width, height, required resolution |
| negative_constraints | text array | Standing print constraints |
| production_notes | text | Garment compatibility, ink coverage |
| edited_by_user | boolean | |
| prompt_version | text | |
| created_at | timestamp | |

**Constraints:** unique on (concept_id, version).

**Reasoning for typography by class rather than font name.** Naming a specific font invites two problems: licensing implications, and models confidently inventing fonts that do not exist. Specifying a class — "condensed slab serif, heavy weight, uppercase, arched" — is both safer and more reliably interpreted.

**Reasoning for versioning briefs.** The operator can edit a brief before generation. Regenerating with a modified brief must not lose the original, because comparing outputs from different briefs is how the operator learns what works.

---

### `generated_artwork`

| Field | Type | Notes |
|---|---|---|
| **id** (PK) | UUID | |
| workspace_id (FK) | UUID → workspaces | |
| concept_id (FK) | UUID → design_concepts | |
| brief_id (FK) | UUID → artwork_briefs | |
| variant_index | integer | |
| status | enum | `queued` / `generating` / `generated` / `processing` / `qa_failed` / `ready` / `accepted` / `rejected` / `failed` |
| provider | text | |
| provider_request_id | text | |
| prompt_text / negative_prompt | text | Exactly what was sent |
| seed | bigint | Reproducibility |
| parameters | JSON | All provider settings |
| is_vector_ready | boolean | |
| visual_embedding | vector | |
| originality_score | decimal | |
| nearest_competitor_listing_id (FK) | UUID | Closest match found |
| nearest_similarity | decimal | |
| originality_acknowledged_by (FK) | UUID → users | |
| cost_amount / cost_currency | bigint / char(3) | |
| accepted_at | timestamp | |
| rejection_reason | text | |
| created_at / updated_at / deleted_at | timestamp | |

**Reasoning for storing the exact prompt, seed and parameters.** Three needs: reproducing a result, iterating with "same seed, adjusted brief", and diagnosing why a generation failed quality checks. Without these the artwork pipeline is a black box.

---

### `artwork_renditions`

Every derived file, with lineage.

| Field | Type | Notes |
|---|---|---|
| **id** (PK) | UUID | |
| workspace_id | UUID | |
| artwork_id (FK) | UUID → generated_artwork | |
| kind | enum | `original` / `background_removed` / `cropped` / `upscaled` / `print` / `web` / `thumbnail` / `vector` / `proof` |
| derived_from (FK) | UUID → artwork_renditions | Self-referencing lineage |
| file_id (FK) | UUID → files | |
| width / height | integer | |
| resolution_at_target | decimal | Effective print resolution |
| has_transparency | boolean | |
| colour_count | integer | |
| byte_size | bigint | |
| created_at | timestamp | |

**Reasoning for the self-referencing lineage.** Processing is a chain. When quality validation fails, knowing *which* step introduced the problem is essential — a halo appearing after background removal is a different fix from one appearing after upscaling. The tree makes this visible.

---

### `artwork_qa_results`

| Field | Type | Notes |
|---|---|---|
| **id** (PK) | UUID | |
| workspace_id | UUID | |
| artwork_id (FK) | UUID → generated_artwork | |
| rendition_id (FK) | UUID → artwork_renditions | |
| overall | enum | `pass` / `warn` / `fail` |
| criteria | JSON | Per check: measured value, threshold, status, remedy |
| qa_version | text | |
| created_at | timestamp | |

**Reasoning for storing the measured value and the threshold, not just pass/fail.** The requirement is that a failure names the specific problem *and* the specific fix. "Effective resolution is 180 at 15×18 inches; 300 required — upscale or regenerate" requires all three pieces of data to be persisted.

---

## C.8 Commerce

### `print_providers`, `blueprints`, `blueprint_variants`

Synchronised catalogue from the fulfilment provider. **Global reference data**, not workspace-scoped.

**`print_providers`**

| Field | Notes |
|---|---|
| **id** (PK) | UUID |
| provider_external_id | Fulfilment provider's identifier |
| name / location | |
| average_production_days | Observed, not claimed |
| reliability_rating | Derived from observed outcomes |
| synced_at | |

**`blueprints`**

| Field | Notes |
|---|---|
| **id** (PK) | UUID |
| provider_external_id | |
| product_type_id (FK) | → product_types |
| title / brand / model | |
| print_areas | JSON — dimensions and required resolution per placement |
| is_active | |
| synced_at | |

**`blueprint_variants`**

| Field | Notes |
|---|---|
| **id** (PK) | UUID |
| blueprint_id (FK) | → blueprints |
| print_provider_id (FK) | → print_providers |
| variant_external_id | |
| colour / colour_hex / size | |
| **cost_amount / cost_currency** | **Production cost — the foundation of every margin calculation** |
| shipping_first_item_amount | |
| shipping_region | |
| in_stock | |
| synced_at | |

**Reasoning for daily variant synchronisation.** Production costs change. A stale cost silently corrupts every margin figure, every profitability sub-score and every pricing decision — and does so invisibly. A daily refresh with an alert on changes above a threshold is the only defence against a slow, unnoticed erosion of margin across an entire portfolio.

**Reasoning for `reliability_rating` being observed rather than claimed.** A cheap provider with poor reliability is not actually cheap. Once orders flow, production times and problem rates feed this rating, which then influences recommendations.

---

### `product_recommendations`

| Field | Notes |
|---|---|
| **id** (PK) | UUID |
| workspace_id / run_id | |
| blueprint_id / print_provider_id (FK) | |
| demand_score / competition_score / profitability_score | Decimals |
| overall_score | |
| suggested_price_amount / currency | |
| unit_cost_amount | |
| margin_percent | |
| recommended_colours / recommended_sizes | Text arrays |
| reasoning | |
| rank | |
| created_at | |

---

### `product_drafts`

**The convergence point of the entire system** — where concept, artwork, product configuration, pricing and listing content meet.

| Field | Type | Notes |
|---|---|---|
| **id** (PK) | UUID | |
| workspace_id (FK) | UUID → workspaces | |
| run_id (FK) | UUID → research_runs | |
| concept_id (FK) | UUID → design_concepts | |
| artwork_id (FK) | UUID → generated_artwork | |
| blueprint_id (FK) | UUID → blueprints | |
| print_provider_id (FK) | UUID → print_providers | |
| seo_variation_id (FK) | UUID → seo_variations | The chosen listing content |
| fee_model_id (FK) | UUID → fee_models | |
| status | enum | `draft` / `fulfilment_pending` / `fulfilment_ready` / `marketplace_pending` / `marketplace_draft` / `partial` / `review` / `published` / `failed` / `archived` |
| retail_price_amount / currency | bigint / char(3) | |
| free_shipping | boolean | |
| unit_cost_amount | bigint | |
| net_profit_amount | bigint | |
| margin_percent | decimal | |
| artwork_placement | JSON | Position, scale, rotation, print area |
| checklist | JSON | Pre-publish check states |
| idempotency_key | text | |
| published_at / published_by | timestamp / UUID | |
| error | JSON | |
| created_at / updated_at / deleted_at | timestamp | |

**Reasoning for a single convergence table.** Everything the final review screen displays hangs off this row. Modelling it as several loosely-connected entities would make that screen an expensive multi-query assembly and would make the status of "the product" ambiguous. One row, one status, one lifecycle.

---

### `product_variants`

| Field | Notes |
|---|---|
| **id** (PK) | UUID |
| workspace_id | |
| product_draft_id (FK) | → product_drafts |
| blueprint_variant_id (FK) | → blueprint_variants |
| sku | Stable, maps marketplace variant to fulfilment variant |
| is_enabled | |
| price_amount / currency | |
| cost_amount | Snapshot at configuration time |
| margin_percent | |
| is_default | |

**Reasoning for snapshotting cost.** The catalogue cost changes daily. The cost used when this product was priced is a fact about the pricing decision and must not drift.

---

### `seo_variations`

| Field | Type | Notes |
|---|---|---|
| **id** (PK) | UUID | |
| workspace_id | UUID | |
| product_draft_id (FK) | UUID → product_drafts | |
| concept_id / run_id (FK) | UUID | |
| variation_index | integer | 1–10 |
| positioning_axis | enum | `gift` / `audience` / `humour` / `benefit` / `occasion` / `longtail` / `broad` / `seasonal` / `personalisation` / `premium` |
| title | text | Length-constrained |
| description | text | Structured sections |
| tags | text array | Exactly 13, each length-constrained |
| keywords | JSON | Term, rank, evidence, competition index |
| positioning | text | |
| materials | text array | |
| taxonomy_id | integer | |
| quality_score | decimal | |
| quality_breakdown | JSON | |
| validation | JSON | Constraint check results |
| is_selected | boolean | |
| edited_by_user | boolean | |
| prompt_version | text | |
| created_at / updated_at | timestamp | |

**Constraints:** database-level checks enforce the title length limit and the maximum tag count; a partial unique index enforces at most one selected variation per draft.

**Reasoning for enforcing marketplace constraints in the database.** These limits are absolute — the marketplace rejects anything that violates them. Enforcing at the database level means no code path, however unusual, can persist an invalid listing. Application validation catches it earlier and more helpfully; the database constraint guarantees it.

**Reasoning for retaining all ten variations.** Nine are unused at publish time but represent tested, ready alternatives. They are the raw material for later listing optimisation experiments, at zero additional generation cost.

**Reasoning for `edited_by_user`.** Heavy editing signals a weak SEO engine. Tracking it turns a vague impression into a measurable quality metric — and prevents human corrections from polluting evaluation data.

---

### `pricing_snapshots`

| Field | Notes |
|---|---|
| **id** (PK) | UUID |
| workspace_id | |
| product_draft_id (FK) | → product_drafts |
| retail_price_amount / currency | |
| breakdown | JSON — every fee line itemised |
| net_profit_amount | |
| margin_percent | |
| fee_model_id (FK) | → fee_models |
| created_at | |

**Reasoning.** The itemised profit calculation shown at final approval is a record of what the operator was told when they decided to publish. If fees or costs change afterwards, that record must remain intact.

---

## C.9 Publishing

### `integrations` and `integration_credentials`

**`integrations`** — connection metadata, freely readable.

| Field | Notes |
|---|---|
| **id** (PK) | UUID |
| workspace_id (FK) | → workspaces |
| provider | `etsy` / `printify` / `ideogram` / `ai` / `market_data` |
| status | `connected` / `disconnected` / `needs_reauth` / `degraded` |
| external_account_id / external_shop_id | |
| scopes | Text array |
| quota_used / quota_limit / quota_reset_at | |
| last_success_at | |
| last_error | JSON |
| circuit_state / circuit_open_until | |
| connected_at | |

**`integration_credentials`** — the secrets, separated.

| Field | Notes |
|---|---|
| **id** (PK) | UUID |
| integration_id (FK) | → integrations |
| workspace_id | |
| ciphertext | Encrypted credential bundle |
| wrapped_key | Encrypted data key |
| key_version | Supports rotation |
| expires_at / rotated_at | |
| created_at | |

**Constraints:** unique on (workspace_id, provider) for integrations.

**Reasoning for the split.** The health dashboard reads integration status constantly. Keeping ciphertext in a separate table means those reads never touch encrypted material, the credential table can have tighter database-level access grants, and accidental logging of a query result cannot leak a secret.

---

### `printify_products`

Fulfilment-side product records.

| Field | Notes |
|---|---|
| **id** (PK) | UUID |
| workspace_id | |
| product_draft_id (FK) | → product_drafts |
| external_product_id | Fulfilment provider's identifier |
| external_image_id | Uploaded artwork identifier |
| blueprint_id / print_provider_id (FK) | |
| status | |
| mockups_ready | |
| external_payload | JSON — full response for reconciliation |
| linked_to_marketplace | **Critical** — orders will not fulfil without this |
| created_at / updated_at | |

**Reasoning for `linked_to_marketplace` as an explicit field.** The step that associates the fulfilment product with the marketplace listing is easy to omit and produces a silent failure — the listing sells, and nothing is manufactured. Making it an explicit tracked field and a mandatory pre-publish check turns a silent catastrophe into a visible blocker.

---

### `etsy_listings`

Marketplace-side listing records.

| Field | Type | Notes |
|---|---|---|
| **id** (PK) | UUID | |
| workspace_id (FK) | UUID → workspaces | |
| product_draft_id (FK) | UUID → product_drafts | |
| concept_id / artwork_id / run_id / niche_id (FK) | UUID | **Full lineage retained on the listing itself** |
| marketplace_listing_id | bigint | |
| marketplace_shop_id | bigint | |
| state | enum | `draft` / `active` / `inactive` / `sold_out` / `expired` / `removed` |
| title | text | |
| price_amount / currency | bigint / char(3) | |
| tags | text array | |
| url | text | |
| published_at | timestamp | |
| first_sale_at | timestamp | |
| last_synced_at | timestamp | |
| sync_error | JSON | |
| predicted_score | decimal | Snapshot of the prediction at publish |
| created_at / updated_at / deleted_at | timestamp | |

**Constraints:** unique on (workspace_id, marketplace_listing_id) — a hard guarantee against duplicate records.

**Reasoning for denormalising lineage identifiers onto the listing.** Concept, artwork, run and niche are all reachable through the product draft. Storing them directly costs four columns and turns the most common analytical queries — "how do gap-derived designs perform?", "which niche produces the best listings?" — from four-table joins into single-table filters. Given how often these run on the analytics dashboards, the denormalisation is justified.

**Reasoning for `predicted_score` being copied here.** Prediction accuracy analysis compares what was predicted against what happened. Snapshotting the prediction at publish time prevents a later re-scoring from silently rewriting history.

---

### `listing_images`

| Field | Notes |
|---|---|
| **id** (PK) | UUID |
| workspace_id | |
| listing_id (FK) | → etsy_listings |
| product_draft_id (FK) | → product_drafts |
| file_id (FK) | → files |
| source | `mockup` / `artwork` / `upload` |
| position | Display order |
| is_primary | |
| external_image_id | Marketplace identifier |
| uploaded_at | |

---

### `publish_jobs`

**The mechanism that makes publishing recoverable.** One row per atomic publishing operation.

| Field | Notes |
|---|---|
| **id** (PK) | UUID |
| workspace_id | |
| product_draft_id (FK) | → product_drafts |
| operation | `upload_artwork` / `create_product` / `fetch_mockups` / `create_draft` / `upload_images` / `set_inventory` / `link_fulfilment` / `publish` |
| status | `pending` / `running` / `succeeded` / `failed` |
| attempt | |
| **idempotency_key** | Derived from draft, operation and input — unique per workspace |
| request_payload / response_payload | JSON |
| error | JSON |
| started_at / finished_at | |

**Constraints:** unique on (workspace_id, idempotency_key).

**Reasoning — this table is the answer to "never create a duplicate listing".** Publishing is not one action; it is eight. A single monolithic operation that fails halfway is unrecoverable without risking duplication. Decomposing it means:

- Each operation succeeds or fails independently and visibly.
- The idempotency key makes retrying any operation safe.
- A partial failure produces a targeted retry that touches only what failed.
- The database uniqueness constraint is the final backstop — even a completely confused caller cannot create the same listing twice.

---

## C.10 Analytics and learning

### `performance_snapshots`

Immutable, partitioned monthly.

| Field | Notes |
|---|---|
| **id** (PK, composite with captured_at) | UUID |
| workspace_id / listing_id | |
| captured_at (partition key) | |
| views / favourites / orders | |
| revenue_amount / currency | |
| state | Listing state at capture |
| views_delta / orders_delta | Change since previous snapshot |
| conversion_rate | |
| source | |

**Reasoning for storing deltas alongside totals.** Totals are cumulative and easy to fetch; deltas are what actually indicate momentum. Computing deltas at write time, when the previous snapshot is already in hand, is far cheaper than computing them across the series at every read.

---

### `outcome_records`

The learning loop's training data.

| Field | Notes |
|---|---|
| **id** (PK) | UUID |
| workspace_id | |
| listing_id (FK) | → etsy_listings |
| concept_id / run_id (FK) | |
| features | JSON — flattened attribute vector |
| feature_set_version | Which feature definition produced it |
| outcome_window_days | 30 / 60 / 90 |
| views / orders / revenue_amount | |
| age_normalised_percentile | **The target variable** |
| days_to_first_sale | |
| is_success | Top quartile in this workspace |
| computed_at | |

**Constraints:** unique on (listing_id, outcome_window_days, feature_set_version).

**Reasoning for a percentile target rather than raw revenue.** Raw revenue is confounded by price, seasonality and portfolio composition. An age-adjusted percentile within the workspace answers the question that matters — *did this design do better than my other designs?* — and is robust to those confounders.

**Reasoning for `feature_set_version`.** As new attributes are captured, the feature vector changes shape. Versioning it means old outcome records remain interpretable and models can be trained on a consistent definition.

---

### `prediction_records`

| Field | Notes |
|---|---|
| **id** (PK) | UUID |
| workspace_id | |
| concept_id / listing_id (FK) | |
| predicted_score / predicted_band | |
| actual_percentile | Filled once outcomes exist |
| error | |
| scoring_config_id (FK) | |
| evaluated_at / created_at | |

**Reasoning.** This table is what allows the product to be honest about its own accuracy. It is deliberately separate from `concept_scores` because it is an *evaluation* record, joining a prediction to a realised outcome — a different concern with a different lifecycle.

---

### `model_proposals`

| Field | Notes |
|---|---|
| **id** (PK) | UUID |
| workspace_id | |
| base_config_id / proposed_config_id (FK) | → scoring_configurations |
| method | Fitting approach used |
| outcome_count | How much data supported it |
| weight_diff | JSON — exactly what would change |
| backtest_results | JSON — performance on held-back data |
| status | `proposed` / `activated` / `rejected` |
| decided_by / decision_note / decided_at | |
| created_at | |

**Reasoning for proposals being a distinct entity from configurations.** A proposal is a *recommendation with evidence*, not a configuration. Separating them means the operator sees the reasoning and the back-test before anything changes, rejected proposals are retained with their reasoning, and activation is an explicit, reversible decision rather than a silent update.

---

## C.11 Platform

### `files`

Metadata for every stored object. Bytes live in object storage.

| Field | Notes |
|---|---|
| **id** (PK) | UUID |
| workspace_id | |
| bucket / object_key | Storage location |
| content_type / byte_size | |
| **sha256** | Content hash — the deduplication key |
| width / height | For images |
| purpose | `competitor_image` / `artwork` / `mockup` / `export` / `import` |
| created_at / deleted_at | |

**Constraints:** unique on (workspace_id, sha256, purpose).

**Reasoning for content-hash deduplication.** Competitor images repeat heavily — the same mockup template appears across dozens of listings. Deduplicating on content means each is downloaded once, stored once and, crucially, *analysed* once. This is one of the largest cost savings in the system.

---

### `ai_prompt_calls`

Every AI invocation. Partitioned monthly.

| Field | Notes |
|---|---|
| **id** (PK, composite with created_at) | UUID |
| workspace_id / run_id / run_step_id | |
| purpose | Which task |
| capability_tier | `reasoning` / `analysis` / `extraction` / `vision` |
| model_identifier | Resolved model, from configuration |
| prompt_id / prompt_version | |
| input_hash | Cache key |
| was_cache_hit | |
| tokens_in / tokens_out / cached_tokens_in | |
| cost_amount / currency | |
| latency_ms | |
| output_valid | Passed its contract |
| repair_attempts | |
| raw_response_reference | Object storage key, 90-day retention |
| error | JSON |
| created_at (partition key) | |

**Reasoning — this table serves four distinct purposes.**
1. **Cost accounting** at the granularity of run, step and purpose.
2. **Quality monitoring** — contract failure and repair rates are the leading indicators of prompt degradation.
3. **Reproducibility** — prompt version plus input hash plus model identifier explains any historical output.
4. **Cache** — a deterministic-purpose call with a matching input hash need not be repeated.

---

### `provider_calls`

Every non-AI external call. Partitioned monthly.

| Field | Notes |
|---|---|
| **id** (PK, composite with created_at) | UUID |
| workspace_id / run_id / run_step_id | |
| provider / operation | |
| http_status / succeeded | |
| latency_ms / retry_count | |
| cost_amount / currency | |
| request_hash | |
| request_body / response_body | JSON — nulled after 30 days |
| error | JSON |
| correlation_id | |
| created_at (partition key) | |

**Reasoning for short retention on bodies.** Full payloads are invaluable for debugging in the days after a failure and worthless thereafter, while consuming significant storage. Nulling the bodies while retaining the metadata preserves all the analytical value at a fraction of the size.

---

### `audit_log`

Append-only. Seven-year retention. No updates or deletions permitted at the database grant level.

| Field | Notes |
|---|---|
| **id** (PK) | UUID |
| workspace_id | |
| actor_user_id / actor_type | |
| action | `listing.publish`, `legal.override`, `credential.rotate`, … |
| entity_type / entity_id | |
| before_state / after_state | JSON |
| justification | For overrides |
| ip_hash | Hashed, never raw |
| correlation_id | |
| created_at | |

**Always recorded:** authentication events, credential changes, **every publish**, **every legal override**, scoring configuration activation, budget changes, data export, deletion.

---

### `notifications`

| Field | Notes |
|---|---|
| **id** (PK) | UUID |
| workspace_id / user_id | |
| type / severity | |
| title / body / link | |
| entity_type / entity_id | |
| read_at | |
| created_at | |

---

## C.12 Relationship map

```
users ──< workspace_members >── workspaces
                                    │
        ┌───────────────────────────┼───────────────────────────┐
        │                           │                           │
    settings              scoring_configurations           fee_models
                                    │
                                 niches
                                    │
                    ┌───────────────┼───────────────┐
                    │                               │
               sub_niches                     research_runs
                    │                               │
                    │              ┌────────────────┼────────────────┐
                    │              │                │                │
                    │          run_steps      run_events    run_shop_selections
                    │                                              │
                    │                                    competitor_shops
                    │                                              │
                    │                                   competitor_listings
                    │                                        │        │
                    │                          listing_snapshots  listing_analysis
                    │                                    │
                    │              ┌─────────────────────┼─────────────────────┐
                    │              │                     │                     │
                    │     opportunity_reports    success_reports        failure_reports
                    │                                    │                     │
                    │                          success_attributes     failure_attributes
                    │
              coverage_cells ──► market_gaps
                                       │
                                       ▼
                              design_concepts ──< concept_scores
                                       │
                          ┌────────────┼────────────┐
                          │            │            │
                 legal_screenings  artwork_briefs   │
                          │            │            │
                   legal_matches   generated_artwork
                                       │        │
                        artwork_renditions   artwork_qa_results
                                       │
                                       ▼
    blueprints ──► product_drafts ◄── seo_variations
         │              │    │
  blueprint_variants    │    └──► pricing_snapshots
         │              │
   product_variants     ├──► printify_products
                        ├──► publish_jobs
                        └──► etsy_listings ──< listing_images
                                   │
                        ┌──────────┼──────────┐
                        │                     │
             performance_snapshots      outcome_records
                                              │
                                     prediction_records
                                              │
                                       model_proposals
                                              │
                                    scoring_configurations
                                        (the loop closes)
```

---

## C.13 The lineage guarantee

Any published listing must be traceable back to the research that produced it, with every link navigable:

```
etsy_listing
  → product_draft
      → generated_artwork → artwork_brief → design_concept
      → seo_variation
      → blueprint + blueprint_variants
  → design_concept
      → market_gap → coverage_cell → sub_niche
      → success_attributes → success_report
      → legal_screening → legal_matches
  → research_run → niche
  → performance_snapshots → outcome_records
```

**This is a functional requirement, not a nicety.** It is what allows the operator to answer "why does this listing exist?" for anything in their shop, and it is what makes the learning loop possible. A nightly consistency check verifies no link in this chain is broken.

---

## C.14 Partitioned and high-volume tables

| Table | Partitioning | Retention |
|---|---|---|
| `listing_snapshots` | Monthly by capture date | 180 days raw, then archived to columnar storage; derived analysis retained indefinitely |
| `performance_snapshots` | Monthly by capture date | Indefinite; downsampled to weekly after one year |
| `ai_prompt_calls` | Monthly | Metadata indefinite; raw responses 90 days |
| `provider_calls` | Monthly | Metadata indefinite; bodies 30 days |
| `run_events` | Monthly | 90 days |

**Projected volumes**

| Table | Year 1, single user | 1,000 workspaces |
|---|---|---|
| `listing_snapshots` | ~1.2 M rows | ~1.2 B rows |
| `listing_analysis` | ~400 K rows | ~400 M rows |
| `ai_prompt_calls` | ~300 K rows | ~300 M rows |
| `provider_calls` | ~500 K rows | ~500 M rows |
| `design_concepts` | ~20 K rows | ~20 M rows |
| `generated_artwork` | ~8 K rows | ~8 M rows |

---

## C.15 Indexing strategy

**Every composite index leads with `workspace_id`**, because every query filters on it.

| Access pattern | Index |
|---|---|
| Run list and filtering | (workspace_id, status, created_at desc) |
| Runs for a niche | (workspace_id, niche_id, created_at desc) |
| Step scheduling | (run_id, step_order); partial index on active statuses |
| Listing analysis for a run | (workspace_id, run_id, estimated_monthly_sales desc) |
| Cohort assignment | (run_id, estimated_monthly_sales desc) |
| Ranked findings | (success_report_id, rank) filtered to sufficient-evidence rows |
| Concept board | (workspace_id, run_id, origin) |
| Drafts awaiting review | Partial index on review and partial statuses |
| Performance series | (workspace_id, listing_id, captured_at desc) |
| Cost reporting | (workspace_id, created_at desc) including cost |
| Visual similarity | Approximate-nearest-neighbour index on the embedding column |
| Text similarity | Approximate-nearest-neighbour index on concept embeddings |
| Fuzzy niche matching | Trigram index on normalised name |
| Full-text search | Generated search vector with inverted index |

**Partial indexes are used deliberately** where queries always filter — active runs, non-deleted rows, sufficient-evidence findings. They are substantially smaller and faster than full indexes on the same columns.

---

## C.16 Migration policy

**Expand and contract, always.** Every schema change must leave the previously deployed application version working:

```
Release N     add a nullable column
Release N     backfill in batches, as a monitored job — never inside the migration
Release N+1   application writes both old and new
Release N+2   application reads new only
Release N+3   remove the old column
```

| Rule | Reason |
|---|---|
| Indexes built concurrently | Avoids locking a table that is being read |
| Lock and statement timeouts on all schema changes | A migration that would block should fail fast, not take the application down |
| Backfills are jobs, not migrations | Resumable, rate-limited, observable |
| Every migration tested against production-shaped data | Timing recorded; anything slow is flagged before it reaches production |
| Partitions managed by a scheduled task | Not by migrations |

---

# Part D — API Design

## D.1 API principles

Nine rules that apply to every endpoint in the system.

| # | Rule | Reason |
|---|---|---|
| 1 | **Workspace scope is never accepted from the client** | It is resolved server-side from the session. This eliminates the entire class of tenant-isolation bugs before multi-user exists. |
| 2 | **Every input is validated against a schema at the boundary** | Internal code never sees an unvalidated shape. Explicit maximum lengths and array sizes on every field. |
| 3 | **Every entity identifier is re-resolved against the workspace** | Passing another workspace's identifier returns *not found*, never *forbidden* — the API must never confirm that another workspace's data exists. |
| 4 | **Every spending endpoint checks the budget before spending** | Not after. A budget discovered to be exceeded after the money is gone is not a budget. |
| 5 | **Every side-effecting external operation requires an idempotency key** | Retries must be safe. |
| 6 | **Every list endpoint is cursor-paginated** | Offset pagination is prohibited on anything that can exceed a few thousand rows. |
| 7 | **Errors are translated, never passed through** | A raw provider error is logged, never returned as the user-facing message. |
| 8 | **Endpoints are thin** | Validate, delegate to a service, serialise. Business logic in a handler is a defect. |
| 9 | **Every mutation declares what it invalidates** | Cache correctness is reviewable in one place. |

---

## D.2 Endpoint classes

Rather than repeating security requirements on every endpoint, endpoints are assigned a class that carries them.

| Class | Requirements |
|---|---|
| **Public** | Rate limited by network address. Authentication endpoints only. No workspace context. |
| **Authenticated** | Valid session required. Workspace and user injected server-side. Read-only. |
| **Mutating** | Authenticated, plus cross-site request forgery protection, plus a transaction, plus an audit entry. |
| **Spending** | Mutating, plus a budget check before any external call, plus cost recording after. |
| **Idempotent** | Mutating or Spending, plus a required idempotency key with stored-response replay for 24 hours. |
| **Elevated** | Mutating, plus recent re-authentication. Used for credentials, legal overrides, budget increases, publishing. |
| **Administrative** | Elevated, plus owner role. |

---

## D.3 Authentication APIs

### Purpose
Establish and manage operator identity and sessions. Manage the connection of third-party accounts.

| Endpoint | Purpose | Inputs | Outputs | Class |
|---|---|---|---|---|
| `auth.login` | Verify credentials, begin second-factor challenge | Email, password | Challenge token, second-factor requirement | Public |
| `auth.verifySecondFactor` | Complete authentication | Challenge token, one-time code | Session established (cookie), user and workspace summary | Public |
| `auth.logout` | End the current session | — | Confirmation | Authenticated |
| `auth.session` | Current session state | — | User, workspace, role, expiry | Authenticated |
| `auth.enrolSecondFactor` | Begin second-factor setup | — | Provisioning secret, recovery codes (shown once) | Elevated |
| `auth.confirmSecondFactor` | Verify enrolment | One-time code | Confirmation | Elevated |
| `auth.changePassword` | Update password | Current password, new password | Confirmation; all other sessions revoked | Elevated |
| `auth.listSessions` | Active sessions | — | Sessions with device, address, last used | Authenticated |
| `auth.revokeSession` | End a specific session | Session identifier | Confirmation | Elevated |
| `auth.useRecoveryCode` | Second-factor recovery | Email, password, recovery code | Session established; code consumed | Public |

### Security requirements

| Requirement | Detail |
|---|---|
| Rate limiting | Ten attempts per minute per network address; thirty per hour |
| Account lockout | Exponential after five failures, from one minute to one hour |
| No user enumeration | Identical response and timing whether or not the account exists |
| Password storage | Memory-hard hashing with versioned parameters supporting upgrade-on-login |
| Password policy | Minimum length enforced, checked against a breached-password list using a privacy-preserving range query |
| Second factor | Mandatory. Time-based codes with a narrow drift window. Recovery codes are single-use and hashed. |
| Session tokens | Opaque random values, hashed at rest, transmitted in cookies marked secure, HTTP-only and same-site |
| Session rotation | On login and on any privilege change |
| Session expiry | Idle expiry of seven days, absolute expiry of thirty days |
| Recovery codes | Displayed exactly once, never retrievable |

---

### Integration connection endpoints

| Endpoint | Purpose | Inputs | Outputs | Class |
|---|---|---|---|---|
| `integration.list` | Connection health for all providers | — | Per provider: status, scopes, quota, last success, last error, circuit state | Authenticated |
| `integration.beginMarketplaceAuth` | Start marketplace authorisation | Provider | Redirect target, opaque state value | Elevated |
| `integration.completeMarketplaceAuth` | Finish authorisation | Authorisation code, state | Connection confirmed, shop metadata | Elevated |
| `integration.connectWithToken` | Connect a token-based provider | Provider, token | Validation result, available shops | Elevated |
| `integration.selectShop` | Choose the shop to operate on | Provider, shop identifier | Confirmation | Elevated |
| `integration.test` | Verify a connection is live | Provider | Result with the specific failing check if unhealthy | Authenticated |
| `integration.disconnect` | Remove a connection | Provider, confirmation | Confirmation; credentials destroyed, history retained | Elevated |

**Security requirements specific to these endpoints:**

- The authorisation flow uses proof-key exchange; the verifier is stored server-side against the opaque state value with a ten-minute lifetime.
- The state value is verified on callback; any mismatch aborts without any token exchange.
- Tokens are encrypted before storage and **never returned to the client**. A masked representation is served from a separate endpoint that does not decrypt.
- Only the minimum necessary permissions are requested, and the requested set is displayed to the operator before redirect.
- Disconnection destroys credentials but retains all listings, artwork and history.

---

## D.4 Research APIs

### Purpose
Create, monitor, control and inspect research runs — the central workflow of the product.

| Endpoint | Purpose | Inputs | Outputs | Class |
|---|---|---|---|---|
| `research.estimate` | Pre-flight cost and duration | Niche, product type, depth | Estimated duration and cost, data sources that will be used and their health, budget headroom | Authenticated |
| `research.createRun` | Start a research run | Niche (identifier or name), product type, style preference, depth, optional budget, seed keywords, excluded terms | Run identifier, initial status, step list, estimate | **Spending + Idempotent** |
| `research.getRun` | Full run state | Run identifier | Run with every step: status, attempt, duration, cost, output pointer, error | Authenticated |
| `research.listRuns` | Run history | Cursor, limit, filters (niche, product type, status, date range) | Paginated runs with summary scores | Authenticated |
| `research.streamProgress` | Live progress | Run identifier, last received event | Event stream (see below) | Authenticated |
| `research.cancelRun` | Stop a run | Run identifier | Confirmation; spending stops, completed work retained | Mutating |
| `research.pauseRun` / `resumeRun` | Suspend and continue | Run identifier | Updated status | Mutating |
| `research.retryStep` | Retry a failed step | Run identifier, step key | Step requeued | Mutating |
| `research.skipStep` | Skip an optional step | Run identifier, step key | Confirmation with the degradation this causes | Mutating |
| `research.raiseBudget` | Increase a run's budget | Run identifier, new amount | Updated budget; run resumes if paused | **Elevated** |
| `research.cloneRun` | Re-run with modifications | Run identifier, overrides | New run identifier | Spending + Idempotent |
| `research.upgradeDepth` | Deepen an existing run | Run identifier, target depth | New run reusing existing data; only the difference is fetched | Spending + Idempotent |
| `research.compareRuns` | Difference between two runs | Two run identifiers | Score movement, competitors entered and exited, factor changes | Authenticated |

### Progress stream contract

Transport is Server-Sent Events. Every event carries an identifier so a dropped connection resumes rather than restarts.

| Event | Payload |
|---|---|
| `step.started` | Step key, timestamp |
| `step.progress` | Step key, percentage, specific activity message, optional data |
| `step.completed` | Step key, duration, cost, pointer to what was produced |
| `step.failed` | Step key, translated error, whether it is retriable |
| `run.status` | New status, cumulative spend, budget |
| `partial.result` | Result kind and payload — allows the interface to render sub-niches while competitor collection is still running |
| `heartbeat` | Timestamp, every fifteen seconds |

**Contract requirements:** progress events are coalesced to at most two per second per step so a chatty worker cannot flood the client. Backpressure is handled server-side. The client falls back to polling after two failed reconnections.

### Security requirements

- Run creation is rate limited per workspace and subject to a concurrent-run ceiling.
- The idempotency key prevents an accidental double-submission creating two paid runs.
- Budget is verified before the run is queued and again before each step.
- Budget increases require recent re-authentication, because they authorise spending.
- The progress stream verifies workspace ownership of the run on connection and on every reconnection.
- Cancellation is authorised for any run in the workspace and takes effect within ten seconds.

---

## D.5 Competitor analysis APIs

### Purpose
Expose the collected market data and the analytical conclusions drawn from it.

| Endpoint | Purpose | Inputs | Outputs | Class |
|---|---|---|---|---|
| `analysis.opportunityReport` | The niche verdict | Run identifier, optional version | Overall score, verdict, five sub-scores with inputs and confidence, seasonality profile, executive summary, degradation state | Authenticated |
| `analysis.subNiches` | Ranked segments | Run identifier | Sub-niches with scores, demand, competition, example terms, evidence sources | Authenticated |
| `analysis.shops` | Selected competitor shops | Run identifier, sort, filters | Shops with metrics, selection score, rank and the criteria they passed | Authenticated |
| `analysis.listings` | Competitor listing table | Run identifier, cursor, limit, sort, filters, requested columns | Paginated listings with all attributes, plus facet counts for filtering | Authenticated |
| `analysis.listingDetail` | One listing in full | Listing identifier | Full metadata, extracted visual analysis, observation history, source record | Authenticated |
| `analysis.distributions` | Chart data | Run identifier, dimension | Binned distributions with cohort overlay | Authenticated |
| `analysis.successReport` | What works here | Run identifier | Cohort definition, ranked findings, synthesis card, correlations, interactions, resolved style with evidence | Authenticated |
| `analysis.failureReport` | What fails here | Run identifier | Cohort definition, ranked anti-findings with causality labels, crowded-loser patterns | Authenticated |
| `analysis.findingEvidence` | The listings behind a finding | Finding identifier | The specific listings supporting it | Authenticated |
| `analysis.gaps` | Ranked opportunities | Run identifier | Gaps with scores, demand and supply evidence, suggested angles, caution flags | Authenticated |
| `analysis.coverageMatrix` | The full supply/demand map | Run identifier | Every cell examined, including those excluded by the demand floor, with the floor value | Authenticated |
| `analysis.keywords` | Keyword statistics | Run identifier, cursor, sort | Terms with frequency, relevance, sales-weighted score, top-decile usage | Authenticated |
| `analysis.export` | Report export | Run identifier, report type, format | Signed download link | Authenticated |

### Contract requirements

**Every finding returned by `successReport` or `failureReport` must include:** cohort prevalence, **baseline prevalence**, lift, sample size and confidence. The response schema makes these required fields, so it is structurally impossible for a client to receive a prevalence figure without its baseline. This is the API-level enforcement of the product's central integrity requirement.

**Findings below the evidence threshold** are returned in a separate collection marked as insufficient evidence — present so the operator can see they were considered, but never mixed into the ranked list.

**Every gap returned must include** demand evidence and supply count. The coverage matrix endpoint returns excluded cells *with the reason for exclusion*, so the demand floor is verifiable rather than a hidden filter.

### Pagination

Cursor-based, using a stable key of sort value plus identifier. This survives concurrent inserts, which offset pagination does not. Total counts on large sets are returned as approximations, and the response says so.

### Security requirements

- All endpoints verify the run belongs to the requesting workspace.
- Filter and sort parameters are validated against an allowed set — never interpolated into a query.
- Requested column sets are validated against a whitelist.
- Export generates a signed link valid for a short period, not a direct object path.
- Export volume is rate limited to prevent bulk extraction.

---

## D.6 AI generation APIs

### Purpose
Trigger and control the generative stages: concepts, briefs and artwork. These are the primary spending endpoints in the system.

| Endpoint | Purpose | Inputs | Outputs | Class |
|---|---|---|---|---|
| `generate.concepts` | Produce concepts for a run | Run identifier | Twenty concepts with scores and reasoning; cost incurred | **Spending + Idempotent** |
| `generate.regenerateConcept` | Replace one concept | Concept identifier, optional steering text | Replacement concept; cost | Spending |
| `generate.regenerateAllConcepts` | Replace the whole set | Run identifier, optional steering, confirmation | New set; prior set retained as a version; cost | Spending + Idempotent |
| `generate.expandConcept` | Variants of one concept | Concept identifier, variation axis, count | Variants with scores; cost | Spending |
| `generate.artworkBrief` | Produce a brief | Concept identifier | Brief with palette, typography direction, constraints, dimensions | Spending |
| `generate.artwork` | Render artwork | Concept identifier, brief identifier, variant count, optional steering | Artwork identifiers, estimated cost | **Spending + Idempotent** |
| `generate.regenerateArtwork` | New variants | Artwork identifier, steering | New variant identifiers; cost | Spending |
| `generate.costEstimate` | What would this cost | Operation, parameters | Estimated cost, budget remaining, whether it would exceed | Authenticated |

### The legal gate — the most important security requirement in this section

**`generate.artwork` must verify, in the service layer, that the concept's most recent legal screening permits generation.** Specifically:

| Screening state | Result |
|---|---|
| No screening exists | **Rejected.** No provider call made. |
| Risk level `blocked` | **Rejected.** No provider call made. No override possible. |
| Risk level `high` without a recorded override | **Rejected.** No provider call made. |
| Risk level `medium` without acknowledgement | **Rejected.** |
| Risk level `none` or `low`, or `medium`/`high` with a valid recorded override | Proceeds |

This check happens **where the external call is made**, not in the interface. A client that bypasses the user interface entirely — a direct call, a replayed request, a bug — still cannot generate artwork for a blocked concept. The disabled button in the interface is a convenience; this is the control.

### Security requirements

| Requirement | Detail |
|---|---|
| Budget | Checked before any provider call. Reserved, then reconciled with actual cost. |
| Idempotency | Required on concept and artwork generation. A double-submission cannot produce two paid generations. |
| Rate limiting | Per workspace, tighter than general mutation limits, because these are the expensive operations. |
| Concurrency | Artwork generation runs at low concurrency deliberately, limiting the blast radius of a runaway loop. |
| Retry ceiling | Automatic retries are capped; beyond that requires explicit operator confirmation. |
| Cost disclosure | Every response includes the cost incurred; every estimate endpoint is available before committing. |
| Prompt isolation | No client input is ever passed directly into a prompt. Steering text is treated as untrusted, length-capped and quarantined. |

---

## D.7 Design APIs

### Purpose
Manage concepts through selection, legal clearance and artwork acceptance — the three-stage creative workflow.

### Concepts

| Endpoint | Purpose | Inputs | Outputs | Class |
|---|---|---|---|---|
| `design.listConcepts` | Concept board | Run identifier, sort, filters | Concepts with scores, origin, risk state, selection state | Authenticated |
| `design.getConcept` | One concept in full | Concept identifier | Full content, score breakdown with contributions, cited evidence, near-duplicate warnings, regeneration history | Authenticated |
| `design.selectConcepts` | Choose what to make | Run identifier, concept identifiers | Count selected, estimated artwork cost, screening jobs queued | Mutating + Idempotent |
| `design.deselectConcept` | Remove from selection | Concept identifier | Confirmation | Mutating |
| `design.createManualConcept` | Operator's own idea | Run identifier, concept content | Concept with scores; screening queued | Mutating |
| `design.archiveConcept` | Set aside | Concept identifier | Confirmation | Mutating |

### Legal screening

| Endpoint | Purpose | Inputs | Outputs | Class |
|---|---|---|---|---|
| `legal.getScreening` | Screening result | Subject type, subject identifier | Risk level, extracted entities, every match with owner, number, class, jurisdiction and link, rationale, safer alternatives | Authenticated |
| `legal.acknowledgeMediumRisk` | Accept medium risk | Screening identifier | Confirmation; recorded | Mutating |
| `legal.overrideHighRisk` | Accept high risk | Screening identifier, typed confirmation phrase, written justification | Confirmation; permanently recorded with actor and reason | **Elevated** |
| `legal.acceptAlternative` | Take a safer version | Screening identifier, alternative index | Concept replaced; re-screened automatically | Mutating |
| `legal.rescreen` | Re-run after an edit | Subject type, subject identifier | New screening | Spending |
| `legal.blockedTerms` | Workspace block list | — | Terms permanently blocked for this workspace | Authenticated |
| `legal.addBlockedTerm` | Add to the block list | Term, reason | Confirmation | Elevated |

**The override endpoint has the strictest requirements in the API:** recent re-authentication, an exactly-matching typed confirmation phrase, a non-empty written justification, and an immutable audit entry recording actor, timestamp, justification and the full screening state at the moment of override. Blocked concepts have no override path at all — the endpoint rejects them unconditionally.

### Artwork

| Endpoint | Purpose | Inputs | Outputs | Class |
|---|---|---|---|---|
| `artwork.listByConcept` | Variants for a concept | Concept identifier | Artwork with status, quality result, originality score, cost | Authenticated |
| `artwork.get` | One artwork in full | Artwork identifier | All renditions with lineage, generation parameters, quality report, originality comparison | Authenticated |
| `artwork.getBrief` / `updateBrief` | Read and edit the brief | Brief identifier, content | Brief; editing creates a new version | Authenticated / Mutating |
| `artwork.removeBackground` | Reprocess transparency | Artwork identifier, method | New rendition | Spending |
| `artwork.upscale` | Increase resolution | Artwork identifier, target dimensions | New rendition | Spending |
| `artwork.autoCrop` | Crop to content | Artwork identifier, padding | New rendition | Mutating |
| `artwork.vectorise` | Produce a vector version | Artwork identifier | Vector rendition, or a stated reason it was skipped | Spending |
| `artwork.runQualityCheck` | Re-validate | Artwork identifier | Per-criterion result with measured value, threshold and remedy | Mutating |
| `artwork.originality` | Similarity report | Artwork identifier | Nearest competitor image with similarity, nearest own prior work | Authenticated |
| `artwork.acknowledgeOriginality` | Accept a similarity warning | Artwork identifier | Recorded | Mutating |
| `artwork.accept` | Choose the winner | Artwork identifier | Status advanced; product step unlocked | Mutating |
| `artwork.reject` | Discard | Artwork identifier, reason | Confirmation | Mutating |
| `artwork.upload` | Operator's own file | Concept identifier, upload reference | Artwork entering at the validation stage | Mutating |
| `artwork.download` | Get a file | Artwork identifier, rendition kind | Signed link | Authenticated |

**A hard contract requirement:** `artwork.accept` rejects any artwork whose most recent quality check did not pass, and any artwork with an unacknowledged originality warning. This is enforced in the service layer, so an artwork that fails validation cannot reach a product draft by any path.

---

## D.8 SEO APIs

### Purpose
Generate, evaluate, edit and select listing content.

| Endpoint | Purpose | Inputs | Outputs | Class |
|---|---|---|---|---|
| `seo.generate` | Produce ten variations | Product draft identifier | Ten variations with content, keywords with evidence, quality scores, validation state | **Spending + Idempotent** |
| `seo.list` | All variations for a draft | Product draft identifier | Variations ranked by quality, with positioning axis labels | Authenticated |
| `seo.get` | One variation in full | Variation identifier | Full content, keyword evidence, quality breakdown, validation detail | Authenticated |
| `seo.regenerateOne` | Replace one variation | Variation identifier, optional steering | Replacement; others untouched | Spending |
| `seo.update` | Manual editing | Variation identifier, changed fields | Updated variation with live validation result | Mutating |
| `seo.validate` | Check without saving | Variation content | Per-field validation with specific violations | Authenticated |
| `seo.select` | Choose the active variation | Product draft identifier, variation identifier | Confirmation; previous selection cleared | Mutating |
| `seo.keywordEvidence` | Why this keyword | Variation identifier, term | The competitor listings using it and their performance | Authenticated |
| `seo.preview` | Rendered preview | Variation identifier | Search-result and listing-page representations | Authenticated |

### Validation contract

Validation is returned as structured per-field results, not a boolean. Each violation names the field, the rule, the current value and the limit — so the interface can highlight the exact tag that is too long rather than saying "invalid".

Rules enforced: title length, exact tag count, individual tag length, no duplicate tags, no excluded terms, no restricted marketplace terms, minimum description length.

### Security requirements

- Generated and edited text passes through restricted-term screening before it can be attached to a draft.
- Editing endpoints validate on write; the database additionally constrains title length and tag count so no path can persist invalid content.
- Only one variation may be selected per draft, enforced by a database constraint rather than application logic alone.

---

## D.9 Product and pricing APIs

### Purpose
Configure the physical product and solve its economics.

| Endpoint | Purpose | Inputs | Outputs | Class |
|---|---|---|---|---|
| `product.recommendations` | Ranked configurations | Run identifier, product type | Configurations with demand, competition and profitability sub-scores, costs, suggested prices, margins, recommended colours | Authenticated |
| `product.createDraft` | Begin a product | Concept identifier, artwork identifier | Draft identifier | Mutating + Idempotent |
| `product.getDraft` | Full draft state | Draft identifier | Everything the review screen needs, plus checklist state | Authenticated |
| `product.listDrafts` | Drafts in progress | Cursor, filters | Paginated drafts with status | Authenticated |
| `product.selectConfiguration` | Choose blueprint and provider | Draft identifier, blueprint, provider | Available variants with costs | Mutating |
| `product.selectVariants` | Choose colours and sizes | Draft identifier, variant identifiers | Variants configured, with artwork-compatibility warnings | Mutating |
| `product.price` | Solve pricing | Draft identifier, mode (target margin or fixed price), value, free shipping | Price, itemised cost breakdown, net profit, margin, whether the floor is met, minimum price to meet it, competitor price context | Mutating |
| `product.updatePlacement` | Adjust artwork position | Draft identifier, position, scale, rotation | Confirmation; preview regenerated | Mutating |
| `product.checklist` | Pre-publish state | Draft identifier | Every check with pass, warn or fail, and a jump target to fix it | Authenticated |
| `product.archiveDraft` | Abandon a draft | Draft identifier | Confirmation | Mutating |

### The pricing contract

The response is deliberately verbose because the operator is making a financial decision:

| Field | Purpose |
|---|---|
| Itemised breakdown | Every cost line separately: production, shipping, listing fee, transaction fee, payment processing, advertising fee, tax |
| **Two profit figures** | With advertising fees charged and without. The margin floor is evaluated against the *with* case. |
| Floor status | Whether the floor is met, and the exact price that would meet it |
| Competitor context | Where this price sits in the observed distribution |

**The floor is enforced, not advised.** A configuration below the floor cannot be selected without an explicit override, and publishing is blocked.

---

## D.10 Printify APIs

### Purpose
Build the product in the fulfilment system and retrieve its imagery.

| Endpoint | Purpose | Inputs | Outputs | Class |
|---|---|---|---|---|
| `fulfilment.catalogue` | Available products | Product type, filters | Blueprints with providers, variants, costs, production times, reliability ratings | Authenticated |
| `fulfilment.syncCatalogue` | Refresh the catalogue | Optional scope | Job identifier; runs asynchronously | Mutating |
| `fulfilment.uploadArtwork` | Send artwork | Draft identifier | External image identifier; deduplicated by content hash | Spending + Idempotent |
| `fulfilment.createProduct` | Build the product | Draft identifier | External product identifier, status | **Idempotent** |
| `fulfilment.updateProduct` | Modify an existing product | Draft identifier, changes | Updated status; never creates a second product | Idempotent |
| `fulfilment.mockupStatus` | Are images ready | Draft identifier | Status, available mockups | Authenticated |
| `fulfilment.selectMockups` | Choose and order images | Draft identifier, ordered identifiers, primary | Confirmation | Mutating |
| `fulfilment.linkToMarketplace` | Associate for order routing | Draft identifier | Confirmation | **Idempotent** |
| `fulfilment.retryOperation` | Repair a failed step | Draft identifier, operation | Operation requeued | Mutating |

### Contract requirements

**Mockup retrieval never blocks a request.** The interface polls a status endpoint; the actual waiting happens in a background job with exponential backoff. An endpoint that waited for mockup generation would time out and would consume a request thread for minutes.

**`linkToMarketplace` is mandatory and appears on the pre-publish checklist.** Omitting it produces a silent failure in which the listing sells and nothing is manufactured. It is treated as a hard blocking check.

**Every operation is recorded as a separate job with its own idempotency key**, so a partial failure is repairable by operation rather than by starting over.

### Security requirements

- Credentials are fetched per call from the credential service, never held in module scope.
- Rate limiting is applied per workspace with adaptive reduction on rate-limit responses.
- Artwork upload validates that quality checks passed before sending anything.
- Provider validation errors are translated into specific remedies, not passed through raw.

---

## D.11 Etsy APIs

### Purpose
Create marketplace drafts, upload images, configure inventory and — separately and explicitly — publish.

| Endpoint | Purpose | Inputs | Outputs | Class |
|---|---|---|---|---|
| `marketplace.shopInfo` | Connected shop details | — | Shop metadata, shipping profiles, sections, policies | Authenticated |
| `marketplace.taxonomy` | Category tree | Optional search | Categories with attributes | Authenticated |
| `marketplace.createDraft` | Create a draft listing | Draft identifier | Marketplace listing identifier, state confirmed as draft | **Idempotent** |
| `marketplace.uploadImages` | Send listing images | Draft identifier, ordered images | Per-image result; partial success reported explicitly | Idempotent |
| `marketplace.setInventory` | Configure variants | Draft identifier | Confirmation | Idempotent |
| `marketplace.publish` | **Make the listing live** | Draft identifier, typed confirmation | Listing identifier, public link, state, timestamp | **Elevated + Idempotent** |
| `marketplace.syncListing` | Refresh from the marketplace | Listing identifier | Current state, drift detected against local record | Authenticated |
| `marketplace.retryOperation` | Repair a failed step | Draft identifier, operation | Operation requeued; nothing already created is duplicated | Mutating |
| `marketplace.listListings` | Published listings | Cursor, filters | Paginated with performance summary | Authenticated |
| `marketplace.getListing` | One listing with lineage | Listing identifier | Listing, performance series, full lineage chain | Authenticated |

### The publish contract — the most carefully specified endpoint in the system

**Server-side pre-flight, executed on every call regardless of what the client believes:**

1. Re-evaluate **every** hard checklist item from current data.
2. Re-verify legal screening state for the concept, the artwork and the listing text.
3. Re-verify that margin meets the floor at the current price.
4. Confirm the listing exists remotely and is still in draft state.
5. Confirm the fulfilment link exists.
6. Confirm quota headroom in the reserved publishing allocation.

If any check fails, the endpoint returns a *checklist failed* error enumerating exactly which items failed. **The disabled button in the interface is a convenience. This is the control.**

**Duplicate prevention operates at four layers:**

| Layer | Mechanism |
|---|---|
| Request | Required idempotency key with stored-response replay |
| Application | Pre-flight check for an existing marketplace listing identifier on the draft |
| Reconciliation | On an ambiguous timeout, search recent drafts for a matching signature and adopt rather than recreate |
| Database | Uniqueness constraint on workspace plus marketplace listing identifier |

**Draft creation always sets the state to draft.** There is no code path that creates an active listing directly.

### Security requirements

- Publishing requires recent re-authentication. It is irreversible and public.
- Publishing is rate limited separately and more tightly than other mutations.
- A reserved portion of the daily marketplace quota is held back for publishing and performance synchronisation, so heavy research can never make publishing impossible.
- Every publish writes an immutable audit entry containing the actor, the timestamp and the exact payload sent.
- Authorisation loss moves pending operations to a *needs reauthorisation* state — paused and resumable, never failed and discarded.

---

## D.12 Analytics APIs

### Purpose
Report on outcomes, accuracy, cost and the learning loop.

| Endpoint | Purpose | Inputs | Outputs | Class |
|---|---|---|---|---|
| `analytics.portfolio` | Overall performance | Date range, filters | Revenue and order series, publication markers, best and worst performers | Authenticated |
| `analytics.listingPerformance` | One listing over time | Listing identifier, range | Views, favourites, orders, revenue series with event markers | Authenticated |
| `analytics.byNiche` | Per-niche comparison | Date range | Niches with published count, revenue, conversion, average outcome percentile | Authenticated |
| `analytics.byConceptOrigin` | Success-derived versus gap-derived | Date range | Comparison with sample sizes and statistical caveats | Authenticated |
| `analytics.predictionAccuracy` | How good are the predictions | Date range | Predicted against actual, calibration by band, accuracy measures, per-dimension predictive power | Authenticated |
| `analytics.costs` | Spending | Date range, grouping | Spend by activity, provider and period; cost per run, artwork and published product; month-end projection | Authenticated |
| `analytics.pipeline` | Throughput | Date range | Runs started and completed, concepts selected per run, artwork acceptance rate, publish success rate |Authenticated |
| `analytics.lineage` | Trace a listing | Listing identifier | The full chain from listing back to research run, each link navigable | Authenticated |
| `analytics.export` | Data export | Type, range, format | Signed download link | Authenticated |

### Learning endpoints

| Endpoint | Purpose | Inputs | Outputs | Class |
|---|---|---|---|---|
| `learning.activeConfiguration` | Current scoring weights | — | Active configuration with version, source and weights | Authenticated |
| `learning.configurationHistory` | All versions | — | Versions with activation dates and notes | Authenticated |
| `learning.proposals` | Pending improvements | — | Proposals with weight differences and back-test results | Authenticated |
| `learning.getProposal` | One proposal in detail | Proposal identifier | Every weight change with direction and magnitude, back-test performance, outcome count | Authenticated |
| `learning.activateProposal` | Accept an improvement | Proposal identifier, note | New configuration active; previous retained | **Elevated** |
| `learning.rejectProposal` | Decline | Proposal identifier, reason | Archived with reasoning | Mutating |
| `learning.revertConfiguration` | Roll back | Target version, reason | Previous configuration reactivated | **Elevated** |
| `learning.rescoreHistory` | Compare under new weights | Configuration version, scope | Job identifier; produces new score records alongside originals | Mutating |
| `learning.readiness` | Is learning possible yet | — | Outcome count, minimum required, whether fitting is enabled | Authenticated |

**Contract requirements:**

- Activation is never automatic. There is no endpoint that applies a proposal without an explicit operator action.
- Activation requires re-authentication, because it changes how every future decision is scored.
- Reverting is always available; no configuration is ever deleted.
- Re-scoring **never mutates historical scores.** It writes new records tagged with the new configuration version, so the original prediction and the re-scored value coexist.
- The readiness endpoint exists so the interface can honestly say "learning needs 50 outcomes; you have 12" rather than presenting false sophistication.

---

## D.13 File APIs

### Purpose
Move bytes in and out without ever proxying them through the application.

| Endpoint | Purpose | Inputs | Outputs | Class |
|---|---|---|---|---|
| `file.requestUploadUrl` | Get a direct upload target | Purpose, content type, byte size | Signed upload target, file identifier, expiry | Mutating |
| `file.confirmUpload` | Register a completed upload | File identifier | Validated file record, or rejection with reason | Mutating |
| `file.getDownloadUrl` | Get a read link | File identifier | Signed link with short expiry | Authenticated |
| `file.importMarketData` | Import an operator file | File identifier, source, optional mapping | Import summary: accepted, rejected, per-row errors | Mutating |
| `file.getImportMapping` | Suggested column mapping | File identifier | Inferred mapping with confidence per column | Authenticated |
| `file.delete` | Remove a file | File identifier | Soft-deleted; recoverable for thirty days | Mutating |

### Why direct upload and download

Bytes never pass through the application tier. Uploads go straight from the browser to object storage using a signed target; downloads come straight from storage using a signed link. This means:

- Large artwork files do not consume application memory or request time.
- Upload and download bandwidth scale with storage, not with application instances.
- A slow upload cannot occupy a request thread.

### Security requirements

| Requirement | Detail |
|---|---|
| Signed target scope | Restricted to a single object key, a single content type and a maximum size, and expires in fifteen minutes |
| Object keys | Generated server-side from identifiers. **User input never contributes to a storage path.** |
| Type verification | Confirmed by inspecting the file's actual leading bytes on registration, never by trusting the declared type or the extension |
| Size limits | Enforced both in the signed target and on registration |
| Download links | Short-lived, single-purpose, served with headers that prevent the browser from interpreting content |
| Access control | Every request verifies the file belongs to the requesting workspace |
| Storage isolation | Buckets are private with no public access. There is no unsigned path to any object. |
| Import safety | Streamed parsing with row limits; exported files neutralise leading characters that spreadsheet software would interpret as formulas |

---

## D.14 Webhook receivers

Inbound endpoints for provider-initiated events.

| Endpoint | Source | Purpose |
|---|---|---|
| `webhooks/fulfilment` | Printify | Product publish status, mockup readiness, order events |
| `webhooks/marketplace` | Etsy | Listing state changes where supported |
| `webhooks/billing` | Payment provider | Subscription lifecycle — future phase |

**Handling contract, identical for all three:**

1. Verify the signature. Reject immediately if invalid.
2. Verify the timestamp is within a narrow window. Reject replays.
3. Check the event identifier against a recent-events cache. Acknowledge and drop duplicates.
4. Persist the raw event.
5. Enqueue a job.
6. Return success — **within milliseconds.**

**No work happens inside a webhook request.** A handler that does real work will eventually exceed the sender's timeout, causing retries, which compounds load exactly when the system is already struggling.

### Security requirements

- Signature verification uses constant-time comparison.
- The signing secret is stored encrypted and rotatable.
- Endpoints are rate limited despite being authenticated by signature.
- Payloads are size-capped.
- Payload content is treated as untrusted input and validated against a schema before use.

---

## D.15 Error model

A single error shape across every endpoint.

| Field | Purpose |
|---|---|
| Code | Stable, machine-readable identifier |
| Message | Human-readable and safe to display |
| Detail | Additional context, safe to display |
| Field | For validation errors — maps to a specific interface field |
| Retriable | Whether retrying could succeed |
| Retry after | Seconds, where applicable |
| Correlation identifier | Ties the error to server-side logs and traces |

### Error codes

| Code | Meaning |
|---|---|
| `VALIDATION_ERROR` | Input failed schema or business validation |
| `UNAUTHENTICATED` | No valid session |
| `REAUTH_REQUIRED` | Session valid but too old for this action |
| `FORBIDDEN` | Authenticated but not permitted |
| `NOT_FOUND` | Absent, or belongs to another workspace — deliberately indistinguishable |
| `CONFLICT` | State conflict, such as already published |
| `LEGAL_GATE_NOT_PASSED` | Artwork attempted before clearance |
| `CHECKLIST_FAILED` | Publish attempted with failing hard checks |
| `BUDGET_EXCEEDED` | Run or workspace budget exhausted |
| `CONCURRENCY_LIMIT` | Too many active runs |
| `RATE_LIMITED` | Client or provider quota |
| `INTEGRATION_NOT_CONNECTED` | Required provider missing |
| `NEEDS_REAUTH` | Provider authorisation revoked or expired |
| `PROVIDER_UNAVAILABLE` | Circuit breaker open |
| `PROVIDER_ERROR` | Upstream failure, translated |
| `AI_OUTPUT_INVALID` | Model output failed its contract after repair attempts |
| `IDEMPOTENCY_KEY_REUSE` | Same key, different request |
| `REQUEST_IN_PROGRESS` | Same key, still running |
| `INTERNAL_ERROR` | Unexpected |

**Two rules:**

1. **Raw provider messages are never returned as the user-facing message.** They are logged and correlated, and a translated message with a specific remedy is returned instead.
2. **Cross-workspace access returns *not found*, with normalised timing.** The API must never reveal that another workspace's data exists.

---

## D.16 Cross-cutting API behaviours

### Idempotency

Required on: run creation, concept generation, artwork generation, product draft creation, SEO generation, and every fulfilment and marketplace mutation.

| Aspect | Behaviour |
|---|---|
| Same key, same request | The stored response is replayed |
| Same key, different request | Rejected as key reuse |
| Same key, still running | Rejected as in progress, with a retry hint |
| Retention | Twenty-four hours |

### Rate limiting

| Scope | Approach |
|---|---|
| Unauthenticated | Tight, per network address |
| Authenticated reads | Generous, per user |
| Authenticated writes | Moderate, per user |
| Spending operations | Tight, per workspace |
| Concurrent runs | Ceiling per workspace |
| Outbound to each provider | Distributed token bucket shared across all processes, with a reserved allocation for critical operations |

Rate-limited responses include the limit, the remainder, the reset time and a retry-after value.

### Versioning

The internal API is versionless — client and server deploy together. The future public API will be path-versioned with additive changes shipped freely and breaking changes requiring a new version and a long deprecation window.

---

# Part E — Backend Services Architecture

## E.1 What "service" means here

**Services are internal modules with enforced boundaries, not separate deployments.** They run inside the worker process. Each has a single responsibility, a defined input and output contract, and no direct dependency on any other service.

**The rules that make them real boundaries rather than folders:**

| Rule | Consequence |
|---|---|
| A service never imports another service | Coordination happens through the orchestrator and the database |
| A service never contains scoring or statistical arithmetic | It calls the pure domain layer for that |
| A service is the only place adapters and the database meet | Pure logic stays pure; entry points stay thin |
| Every service exposes the same shape: one execute function taking a step context | Uniform retry, timing, cost and error handling |
| A service's output is written transactionally with its status | A step is never "succeeded" without its output persisted |

**Why this discipline matters.** These boundaries are drawn exactly where service boundaries would go if the system were ever split into separate deployments. Extraction later becomes moving a directory and adding a transport, not untangling a monolith.

### Uniform service contract

Every service receives a **step context** containing: workspace identifier, run identifier, step key, attempt number, the active scoring configuration, the budget guard, a cancellation signal, the clock, a correlation identifier, and a logger already bound to that context.

Every service returns a **step output** containing: a pointer to the entity it produced, the cost it incurred, any degradation it experienced, and any warnings for the operator.

---

## E.2 Market Research Service

**Responsibility:** turn a niche name into a structured market picture — sub-niches, demand, competition, trend, profitability and seasonality — and produce the opportunity verdict.

### What it does

| Stage | Detail |
|---|---|
| Niche resolution | Normalise the entered name, match against existing niches by fuzzy similarity, create or reuse |
| Sub-niche discovery | Merge three independent signals: domain reasoning, co-occurrence patterns in observed listing text, and marketplace taxonomy. Deduplicate, record which signals supported each. |
| Demand estimation | Aggregate observed sales, review velocity, engagement and term breadth. Apply the sales-concentration adjustment so a niche dominated by one shop is not mistaken for available demand. |
| Competition assessment | Saturation, incumbent entrenchment, the review moat held by top performers, bestseller density, price compression |
| Trend determination | Slope of review accumulation across snapshots, new-listing rate, new-entrant rate |
| Profitability calculation | Real production costs against the workspace fee model at the observed price point |
| Seasonality profiling | Twelve-month index, peaks, troughs, strength coefficient, publishing lead time |
| Verdict | Weighted combination via the active scoring configuration, mapped to a band, with an executive summary |

### Boundaries

- **Does not** collect competitor data — it consumes what the Competitor Analysis Service produced.
- **Does not** compute any score itself — it assembles features and calls the pure domain layer.
- **Emits a provisional report early** for perceived speed, then a final report once competitor data lands. Both are persisted; neither overwrites the other.

### Degradation behaviour

Every sub-score has a documented fallback. If sales estimates are unavailable, demand falls back to a review-derived proxy with confidence dropped to low and the method named. If snapshot history is insufficient, trend falls back to new-listing and new-entrant rates. **The service never fails because a feature is missing** — it returns a labelled partial result.

---

## E.3 Competitor Analysis Service

**Responsibility:** discover the right shops, collect their relevant listings, and store the result as immutable, provenance-tagged observations.

### Sub-stages

| Stage | Detail |
|---|---|
| Discovery | Query the market data provider chain for candidate shops in the niche and product type |
| Qualification | Apply gates: minimum sales or reviews, minimum relevant listings, currently active |
| Scoring | Combine sales, revenue, reviews, review velocity, listing count and shop age |
| Age preference | Apply the multiplier favouring shops under three years old |
| Selection ladder | Take twenty; fall back to ten, then five; below five, proceed and mark the run degraded |
| Collection | Per shop, gather all listings matching the product type and niche or sub-niche terms, within caps |
| Snapshotting | Write immutable observations with full provenance per field group |
| Asset capture | Download primary images, hash them, deduplicate against existing files |

### Design decisions worth stating

**Failures are isolated per shop.** One shop returning bad data, timing out, or having been deleted must never fail the step. Shops are processed concurrently with per-shop error containment, and the count of failures is reported.

**Truncation is reported, never silent.** If a shop has 900 relevant listings and the cap is 250, the service takes the highest-signal 250 and records that it truncated. A silent cap would corrupt every downstream statistic without any indication.

**Reuse before re-fetch.** Shops and listings observed within the reuse window are used from storage rather than re-fetched. This is the largest cost saving on repeat research in the same niche.

**Selection rationale is persisted.** The interface must be able to answer "why is this shop in my analysis?", which requires storing which criteria each shop passed and what its score was.

---

## E.4 Style Extraction Service

**Responsibility:** convert competitor images into structured visual data. **This is the product's core technical differentiator.**

### Split between deterministic and model-based extraction

| Attribute | Method | Reason |
|---|---|---|
| Colour palette | **Deterministic** — perceptual-space clustering | Measuring colour is more reliable and more reproducible than asking a model to describe it |
| Palette family | **Deterministic** — mapping extracted colours to defined families | Consistent classification across runs |
| Typography style | Model-based vision classification | Requires visual judgement |
| Layout archetype | Model-based | Requires visual judgement |
| Mockup presentation | Model-based | Requires visual judgement |
| Subject matter, humour type | Model-based | Requires semantic understanding |
| Visual embedding | Model-based | Similarity and originality |

**Why colour is deterministic.** Asking a model "what colours are in this image" produces plausible but unreproducible answers that vary between calls. Clustering the actual pixels in a perceptual colour space produces the same answer every time and can be verified. Since palette is one of the most important success attributes the product reports, it must be measured, not described.

### Cost architecture

This service is the single largest AI cost in a research run. Three mechanisms control it:

| Mechanism | Effect |
|---|---|
| **Content-hash caching** | Images are analysed once ever, keyed by content hash and analyser version. On repeat research in a niche, hit rates commonly exceed 40%. |
| **Batching** | Multiple images per model call rather than one each |
| **Deterministic first** | Colour extraction costs nothing and runs before any model call |

### Failure behaviour

An image that fails analysis leaves the listing with no style profile. That listing is retained, excluded from style statistics, and **the exclusion count is reported** — "style analysis available for 284 of 312 listings". Silently analysing 91% of listings while presenting conclusions as though they covered all of them would be a integrity failure.

---

## E.5 Success Analysis Service

**Responsibility:** determine what successful listings in this specific niche have in common, with defensible statistics.

### Process

| Stage | Detail |
|---|---|
| Cohort assignment | Partition listings by sales normalised for listing age, excluding those too new to judge |
| Contingency analysis | For every categorical attribute: prevalence in the cohort, prevalence in the population, ratio, sample size, significance |
| Numeric analysis | For every numeric attribute: cohort and population distributions, effect size, optimal band search |
| Suppression | Findings below sample-size or significance thresholds are stored but flagged and excluded from ranking |
| Weighting | Each finding receives a weight derived from its ratio, confidence and sample size |
| Statement rendering | Findings become sentences via templates, not model generation |
| Synthesis | The "median winning listing" specification |
| Correlation | Rank correlations between numeric attributes and performance, with multicollinearity flagging |
| Interaction detection | Attribute pairs whose combined effect exceeds their individual effects |
| Style resolution | When automatic style selection was requested, determine the winner from measured performance |

### Critical design decisions

**All arithmetic lives in the pure domain layer.** The service assembles data and persists results; it computes nothing itself. This is what makes every statistic reproducible and property-testable.

**Statements are templated, not generated.** A model writing "84% of top listings use muted greens" could produce a number that does not match the stored statistic. Rendering from the computed values guarantees the sentence and the data agree.

**Baseline is a required output field.** The service cannot produce a finding without it, so no downstream consumer can display a prevalence figure without its baseline.

**Supporting listing identifiers are captured at analysis time**, making evidence click-through a single lookup rather than a reconstructed cohort query.

---

## E.6 Failure Analysis Service

**Responsibility:** determine what persistently under-performing listings have in common.

Applies the identical statistical machinery in the inverse direction, with three additions:

| Addition | Purpose |
|---|---|
| **Exposure filtering** | Only listings old enough to have had a fair chance are included. A listing published last week is not a failure. |
| **Causality labelling** | Each finding is classified as causally plausible or correlation-only. "One image" has a mechanism; "uses orange" does not. |
| **Crowded-loser detection** | Attribute combinations that are both widespread and over-represented in failures — the trap most of the market has fallen into |

**Ambiguity handling.** An attribute appearing in both reports is common rather than meaningful. Rather than suppressing it, the service marks it ambiguous, reports the net effect, and reduces its downstream weight.

**Output feeds forward** as negative constraints for concept generation and as penalty terms in design scoring.

---

## E.7 Market Gap Service

**Responsibility:** find underserved segments worth entering.

### Process

| Stage | Detail |
|---|---|
| Coverage mapping | Build the matrix of sub-niche by design angle by style, counting observed supply per cell |
| Demand estimation | Estimate demand per sub-niche independently of supply |
| **Demand floor** | **Exclude cells below the floor entirely** |
| Monetisability | Observed price power and achievable margin in the segment |
| Feasibility | Penalties for trademark-heavy territory, seasonal dead zones, and ideas that cannot be printed |
| Scoring and ranking | Combine into a gap score; rank |
| Explanation | Produce the reasoning and suggested design angles |

### The demand floor is this service's most important behaviour

Without it, the highest-scoring gap is always a cell with zero supply — and zero supply usually means zero demand. A gap engine without a demand floor reliably recommends deserts, and does so with confidence.

**Implementation requirement:** excluded cells are *retained* in the coverage matrix with their exclusion reason, so the visualisation can draw the floor and the operator can verify the logic rather than trusting it.

---

## E.8 Opportunity Report Service

**Responsibility:** assemble the four analyses into the operator-facing verdict.

This service does no analysis of its own. It:

- Combines sub-scores using the active scoring configuration.
- Maps the result to a verdict band.
- Ranks sub-niches by their independent scores.
- Produces the executive summary naming the biggest opportunity and the biggest risk.
- Computes the percentile against the workspace's research history.
- Assembles degradation notices into a single honest statement of what was missing and what would fix it.
- Versions the report so provisional and final coexist.

**Why it is separate from the Market Research Service.** Research gathers and computes; reporting assembles and presents. Keeping them apart means the report can be regenerated under a different scoring configuration — which is exactly what the learning loop's re-scoring feature requires — without re-running any research.

---

## E.9 AI Orchestration Service

**Responsibility:** the single gateway for every model call in the system. No other service calls a model directly.

### What it owns

| Concern | Detail |
|---|---|
| Tier resolution | Capability tier to concrete model, from configuration |
| Prompt rendering | From the versioned registry, with variables |
| Untrusted content quarantine | External text wrapped, delimited and neutralised |
| Cache lookup | By input hash, for deterministic purposes only |
| Budget reservation | Before the call, not after |
| Rate limiting | Per tier, distributed across processes |
| Output contract enforcement | Validation, one repair attempt, one stricter retry, then explicit failure |
| Cost and usage recording | Tokens, cost, latency, cache status, contract validity |
| Circuit breaking | Per tier |

### Why this is centralised

Six services need model calls. If each implemented its own retry, caching, budgeting and validation, there would be six subtly different implementations, six places for a cost leak, and no consistent view of AI usage.

Centralising means: one place to change model bindings, one place to add caching, one complete cost ledger, one uniform failure behaviour, and one place where prompt-injection defences are guaranteed to be applied.

Detailed design in [Part F](#part-f--ai-architecture).

---

## E.10 Design Generation Service

**Responsibility:** produce twenty distinct, evidence-grounded concepts and score them.

### Process

| Stage | Detail |
|---|---|
| Grounding assembly | Build a compact context: top weighted findings, top gaps, resolved style, sub-niche list, excluded terms, prior concept names |
| Parallel generation | Success-derived and gap-derived sets generated concurrently with different grounding |
| Validation | Output checked against its contract |
| Embedding | Each concept embedded for similarity |
| Within-set deduplication | Concepts too similar to a sibling are rejected |
| History deduplication | Concepts too similar to prior workspace concepts are flagged, not silently dropped |
| Quota refill | Regenerate to maintain exactly twenty distinct concepts |
| Scoring | Delegate to the Prediction Service |
| Persistence | Store with prompt version and cited evidence identifiers |

### The originality guarantee, structurally

**The grounding context contains no competitor identifiers, no listing titles, no descriptions and no image references — only aggregate statistical attributes.**

This is not a policy applied to a prompt. It is a property of what the service assembles. There is no path by which a specific competitor design could influence generation, because the specific design never enters the context.

This also means prompt injection through competitor listing text cannot reach the generative stage: the stage that reads external text (style classification) is a different call, with enum-constrained output, and only counts flow forward.

---

## E.11 Prediction Service

**Responsibility:** score concepts and artwork on five dimensions, deterministically.

| Dimension | Basis |
|---|---|
| Market Fit | Weighted alignment with success findings, minus alignment with failure findings |
| Originality | Embedding distance from competitor content and from the workspace's own prior work |
| Conversion | Attributes empirically associated with converting views into purchases |
| Competition | Density of directly comparable existing listings, inverted |
| Opportunity | The gap score of the cell occupied, or the sub-niche score |

**Every dimension is computed in the pure domain layer.** No model produces any of these numbers.

**Two scoring events per concept.** Once as text, once after artwork exists — because originality and conversion signals change when the image is real. Both are retained so the system can later measure which stage predicts outcomes better.

**Reasoning is rendered from the contribution vector**, not authored. This guarantees the explanation always matches the score, which free-form generation cannot.

**The Conversion dimension is where the learning loop lands.** Initially it uses expert-set coefficients. Once sufficient outcomes exist, fitted coefficients replace them — within the same deterministic framework, not by replacing the framework with a model.

---

## E.12 Legal Checking Service

**Responsibility:** assess trademark, copyright and marketplace-policy risk, and gate artwork generation.

### Process

```
extract entities  ──►  internal block list check
                  ──►  registry lookups (parallel, cached)
                  ──►  marketplace restricted-term check
                  ──►  copyright risk assessment
                            │
                            ▼
                  deterministic rule table
                            │
                            ▼
              risk level: none / low / medium / high / blocked
                            │
                    if above low ──► generate safer alternatives ──► re-screen
```

### The most important design decision: the model judges, a rule table decides

A model assesses whether a concept describes a recognisable character, quotes protected material, or references a brand. **It does not assign the risk level.** The risk level comes from an explicit rule table combining the model's assessment with registry results and block-list hits.

**Why this matters:**

| Benefit | Detail |
|---|---|
| Auditable | The decision path can be explained precisely |
| Tunable | Risk appetite changes by editing a table, not a prompt |
| Consistent | Identical inputs always produce identical risk levels |
| Testable | Adversarial cases are table-driven tests, not prompt evaluations |

### Enforcement

The gate is enforced **in the service layer, where the external call is made** — not in the interface. Three subjects are screened: concepts before generation, artwork after generation (for anything the brief could not guarantee, such as invented text or unintended logos), and listing text before publishing.

**Blocked has no override path.** High risk requires re-authentication, an exact typed confirmation, and a written justification, all permanently recorded.

### Caching

Registry lookups are cached by normalised term for thirty days. Trademark registrations do not change hourly, and uncached lookups would make screening both slow and expensive.

---

## E.13 Artwork Service

**Responsibility:** turn an approved concept into validated, print-ready files.

### Pipeline

```
compile brief → compile provider request → generate variants → ingest and hash
   → remove background → refine edges → crop to content → upscale to print size
   → validate print readiness → produce renditions → embed and check originality
   → vectorise where suitable → safety re-screen → ready for acceptance
```

### Why each stage is a separate job

A generation costs real money. If background removal fails, that money must not be lost. Separating stages means:

- A failure late in processing never discards a successful generation.
- Each stage retries independently with its own policy.
- The operator can re-run a single stage — re-upscale, re-remove-background — without regenerating.
- Lineage is explicit, so a defect can be attributed to the stage that introduced it.

### Print-readiness validation

Deterministic, versioned and **blocking**. Each criterion reports the measured value, the threshold, the status and the specific remedy. Artwork that fails cannot be attached to a product draft — enforced in the service, not the interface.

### Provider independence

Only the generation stage is provider-specific. Background removal, cropping, upscaling, validation, renditions, originality checking, vectorisation and safety review are all provider-independent — roughly eighty percent of the pipeline. Swapping providers is an adapter plus style-template mappings plus an evaluation run.

**A hard architectural constraint:** the generation adapter has **no code path that accepts an external image as input**, other than the artwork's own prior renditions for upscaling. Competitor imagery cannot reach the provider by construction.

---

## E.14 Product Service

**Responsibility:** recommend product configurations and solve pricing.

| Concern | Detail |
|---|---|
| Catalogue | Maintain synchronised blueprints, providers, variants and **real production costs** |
| Recommendation | Rank configurations on demand, competition and profitability |
| Colour recommendation | Derived from the success cohort, validated against the artwork's ink |
| Pricing | Solve for target margin or fixed price against the full fee stack |
| Margin floor | Enforce; report the exact price required to meet it |
| Placement | Compute artwork position and scale per product type |

**Advertising fees are modelled as charged by default.** Both figures are produced, and the floor is evaluated against the pessimistic one. Optimistic margin arithmetic is the most common way sellers in this market lose money, and the service is designed not to participate in it.

**Cost is snapshotted at configuration time.** Catalogue costs refresh daily; the cost used for a pricing decision must not drift after the decision was made.

**Cost-change monitoring** is a scheduled responsibility of this service: when a variant's cost moves beyond a threshold, affected unpublished drafts are repriced and the operator is alerted.

---

## E.15 SEO Service

**Responsibility:** produce, validate and score listing content.

| Stage | Detail |
|---|---|
| Keyword pool | Competitor tags weighted by the sales of listings using them, cross-referenced against a general baseline, plus sub-niche terms and operator seeds |
| Generation | Ten variations along ten declared positioning axes |
| Hard validation | Marketplace constraints checked; one automatic repair attempt, then regeneration |
| Quality scoring | Keyword coverage, keyword placement, tag diversity, readability, competitive sensibility |
| Screening | All text passed through restricted-term checking |
| Ranking | By quality score |

**The sales weighting is the differentiator.** Raw term frequency identifies common words; weighting by the performance of the listings using each term identifies effective ones. This is what makes the output evidence-based rather than a frequency counter.

**Competitive sensibility deliberately does not maximise low competition.** A keyword with zero competition is usually a keyword nobody searches. The scoring targets a moderate competition band.

---

## E.16 Printify Service

**Responsibility:** build the product in the fulfilment system and retrieve its imagery.

| Operation | Notes |
|---|---|
| Catalogue synchronisation | Weekly for structure, **daily for costs** |
| Artwork upload | Deduplicated by content hash; never uploaded twice |
| Product creation | With computed placement; idempotent |
| Mockup retrieval | Asynchronous with backoff; **never blocks a request** |
| Mockup selection | Filtered to a preferred set, ordered by what performs in this niche |
| Marketplace linking | Mandatory; tracked explicitly; a blocking pre-publish check |
| Update | Modifies the existing product; never creates a second |

**Reconciliation on ambiguous failure.** If a creation call times out with an unknown outcome, the service searches recent products for a matching signature and adopts it rather than creating a duplicate.

---

## E.17 Etsy Service

**Responsibility:** create drafts, upload images, configure inventory, publish on explicit instruction, and synchronise performance.

### Decomposition

Publishing is **eight independently idempotent operations**, each recorded with its own key and status. This is what makes partial failure repairable and duplicates impossible.

### Rate management

The marketplace enforces both a per-second and a daily limit. The service maintains a distributed token bucket with **a reserved allocation for publishing and performance synchronisation**, so a heavy research day can never leave the operator unable to publish. Priority classes ensure publishing preempts research when contended.

### Draft-only automation

There is **no code path that creates an active listing directly.** Creation always sets draft state; activation is a separate, separately authorised operation with a full server-side pre-flight.

### Synchronisation

Daily per published listing, more frequent in the first week when the signal matters most. Writes immutable snapshots, computes deltas at write time, and detects drift between local and remote state without overwriting either side.

---

## E.18 Analytics Service

**Responsibility:** aggregate outcomes into the dashboards and maintain derived measures.

| Concern | Detail |
|---|---|
| Derived metrics | Computed on write, where the previous snapshot is already at hand |
| Percentile computation | Recomputed nightly across the portfolio |
| Prediction evaluation | Join predictions to realised outcomes, compute accuracy measures |
| Cost aggregation | Roll up spending by activity, provider and period |
| Lineage assembly | Produce the full chain from listing back to research |
| Anomaly detection | First sale, deactivation, zero views after a period, cost spikes |

**Reads route to a replica** where one exists, so analytical queries never compete with the pipeline for the primary database.

---

## E.19 Learning Service

**Responsibility:** convert outcomes into proposed improvements — and refuse to do so when the data does not support it.

| Stage | Detail |
|---|---|
| Feature assembly | Build outcome records linking design and listing attributes to realised performance |
| Readiness check | Below the minimum outcome count, **refuse to fit and say so** |
| Time-based split | Train on earlier data, test on later. Random splits leak future information through shared market conditions and are prohibited. |
| Fitting | Derive candidate weights |
| Shrinkage | Blend toward the existing weights in proportion to sample size |
| Back-testing | Evaluate on held-back data |
| Proposal | Write a proposal with the weight differences and back-test results |
| Notification | Inform the operator |

**The service never activates anything.** Activation is an explicit operator action requiring re-authentication. Every previous configuration is retained and revertible.

**Guards that prevent the loop from doing damage:** a minimum outcome count, a maximum weight movement per proposal, sign constraints on established weights, and a minimum back-test improvement before a proposal is surfaced at all.

---

## E.20 Service interaction map

```
                          ┌──────────────────┐
                          │   ORCHESTRATOR   │
                          │  (step sequencing│
                          │   and budgets)   │
                          └────────┬─────────┘
                                   │ invokes, in dependency order
   ┌───────────────────────────────┼───────────────────────────────┐
   │                               │                               │
┌──▼──────────────┐   ┌────────────▼────────────┐   ┌──────────────▼───┐
│ Market Research │   │  Competitor Analysis    │   │ Style Extraction │
└──┬──────────────┘   └────────────┬────────────┘   └──────────┬───────┘
   │                               │                            │
   │                    ┌──────────┴──────────┬─────────────────┘
   │                    │                     │
   │         ┌──────────▼──────┐   ┌──────────▼──────┐
   │         │ Success Analysis│   │ Failure Analysis│
   │         └──────────┬──────┘   └──────────┬──────┘
   │                    └──────────┬──────────┘
   │                    ┌──────────▼──────────┐
   └───────────────────►│   Market Gap        │
                        └──────────┬──────────┘
                        ┌──────────▼──────────┐
                        │ Opportunity Report  │
                        └──────────┬──────────┘
                        ┌──────────▼──────────┐
                        │ Design Generation   │◄──── AI Orchestration
                        └──────────┬──────────┘
                        ┌──────────▼──────────┐
                        │ Prediction          │
                        └──────────┬──────────┘
                    ═══════════════▼═══════════════
                       HUMAN GATE: selection
                    ═══════════════┬═══════════════
                        ┌──────────▼──────────┐
                        │ Legal Checking      │
                        └──────────┬──────────┘
                    ═══════════════▼═══════════════
                       HUMAN GATE: clearance
                    ═══════════════┬═══════════════
                        ┌──────────▼──────────┐
                        │ Artwork             │◄──── AI Orchestration
                        └──────────┬──────────┘
                    ┌──────────────┼──────────────┐
         ┌──────────▼───────┐          ┌──────────▼──────┐
         │ Product          │          │ SEO             │
         └──────────┬───────┘          └──────────┬──────┘
                    └──────────────┬──────────────┘
                        ┌──────────▼──────────┐
                        │ Printify            │
                        └──────────┬──────────┘
                        ┌──────────▼──────────┐
                        │ Etsy (draft)        │
                        └──────────┬──────────┘
                    ═══════════════▼═══════════════
                       HUMAN GATE: publish
                    ═══════════════┬═══════════════
                        ┌──────────▼──────────┐
                        │ Etsy (publish+sync) │
                        └──────────┬──────────┘
                        ┌──────────▼──────────┐
                        │ Analytics           │
                        └──────────┬──────────┘
                        ┌──────────▼──────────┐
                        │ Learning            │
                        └──────────┬──────────┘
                                   │ proposes new weights
                                   └────────► scoring configuration
                                              (the loop closes)
```

**No arrow in this diagram is a direct service-to-service call.** Every one is the orchestrator invoking the next step once its dependencies are satisfied, with data passed through the database.

---

# Part F — AI Architecture

## F.1 Governing principles

| # | Principle | Consequence |
|---|---|---|
| 1 | **Deterministic arithmetic, model judgement** | No number appearing in any score is produced by a model |
| 2 | **Structured output only** | Every call is bound to an output contract; free text exists only inside a declared field |
| 3 | **Aggregate before prompting** | Statistical summaries, never raw record dumps — the largest cost and quality lever simultaneously |
| 4 | **Cheapest capable tier** | Tier assignment is justified by evaluation results, not intuition |
| 5 | **Untrusted input is quarantined** | External text is data, never instruction |
| 6 | **Everything versioned and logged** | Prompt version, model, input hash, tokens, cost, contract validity — on every call |
| 7 | **Caching is architectural** | Deterministic purposes cached permanently by input hash; creative purposes never cached |

---

## F.2 Where AI is and is not used

| Pipeline stage | Model's role | Deterministic role |
|---|---|---|
| Sub-niche discovery | Domain expansion, naming, rationale | Co-occurrence mining, taxonomy matching, merging, scoring, ranking |
| Opportunity scoring | Executive summary only | **All five sub-scores and the overall score** |
| Shop selection | None | Entirely deterministic |
| Listing collection | None | Entirely deterministic |
| Style extraction | Typography, layout, mockup, subject classification | **Colour palette extraction**, embeddings |
| Success analysis | Optional narrative | **All statistics, ratios, significance, weights, statements** |
| Failure analysis | Causality plausibility judgement | All statistics |
| Gap detection | Angle naming, explanatory prose | Coverage matrix, demand estimation, scoring, ranking, the demand floor |
| Prediction | None | **All five dimensions** |
| Concept generation | **Primary — the creative act** | Deduplication, embedding, quota enforcement |
| Legal screening | Entity extraction, copyright assessment | Registry lookup, block-list matching, **the risk rule table** |
| Artwork brief | **Primary — brief authoring** | Palette derivation, dimension calculation |
| Artwork generation | **Primary — image rendering** | Background removal, upscaling, validation, vectorisation, originality |
| Product recommendation | Explanatory prose | All scoring and pricing arithmetic |
| SEO generation | **Primary — titles, descriptions, tags** | Keyword pool construction, validation, quality scoring |
| Learning loop | None | Fitting, back-testing, shrinkage |

**The pattern:** models do what only models can do — language, vision, creative synthesis. Everything countable is counted.

---

## F.3 Capability tiers

Application code references **capability tiers**. Tier-to-model binding lives in configuration. A model upgrade is therefore a configuration change plus an evaluation run.

| Tier | Characteristics needed | Used for | Approximate share of calls |
|---|---|---|---|
| **Reasoning** | Strongest instruction-following and creative synthesis | Concept generation, artwork briefs, copyright assessment, gap explanations | ~15% |
| **Analysis** | Strong quality at materially lower cost | Sub-niche discovery, SEO, entity extraction, safer alternatives | ~35% |
| **Extraction** | Fast and cheap, high volume | Simple classification, normalisation, short structured transforms | ~35% |
| **Vision** | Multimodal, batchable | Competitor image analysis, artwork safety review | ~15% |

**Tier selection rule:** every new task starts at the extraction tier. It is promoted only when evaluation shows a quality deficit that matters, and the promotion decision is recorded with the measured difference.

**Where the tiers map.** The Claude family provides all four — an Opus-class model for reasoning, a Sonnet-class model for analysis, a Haiku-class model for extraction, and multimodal capability for vision. Exact model identifiers are pinned in environment configuration and never appear in application code.

---

## F.4 The call lifecycle

```
   Engine requests a generation
        │
        ▼
   1. Resolve capability tier → concrete model identifier
        │
   2. Render prompt from the versioned registry with variables
        │
   3. Wrap any untrusted external content in quarantine delimiters
        │
   4. Compute input hash
        │
        ▼
   5. Cache policy permits and hash matches? ──yes──► return cached, record hit
        │ no
        ▼
   6. Budget guard: reserve estimated cost ──denied──► fail with budget error
        │ allowed
        ▼
   7. Rate limiter: acquire slot for this tier
        │
        ▼
   8. Call provider with the output contract and prompt caching enabled
        │
        ▼
   9. Parse and validate against the contract
        │
        ├── valid ──────────────────────────► persist, cache if permitted
        │
        ├── invalid, attempt 1 ──► repair call with the specific errors ──► retry validation
        │
        ├── invalid, attempt 2 ──► stricter retry at zero temperature ──► retry validation
        │
        └── invalid, attempt 3 ──► fail explicitly, preserve raw output for inspection
        │
        ▼
  10. Commit actual cost; record tokens, latency, cache status, contract validity
```

### Reliability settings

| Control | Setting |
|---|---|
| Timeouts | Shorter for extraction and analysis; longer for reasoning with extended thinking and for batched vision |
| Transport retries | Exponential backoff with jitter, honouring any retry hint from the provider |
| Contract repair | One repair, one stricter retry, then explicit failure |
| Circuit breaking | Per tier; an open breaker moves dependent steps to a blocked-external state rather than failing them |
| Concurrency | Per-tier distributed semaphore |
| Token-rate limiting | Per-minute input and output token budgets per tier, enforced before dispatch |
| Determinism | Zero temperature for all classification and extraction; higher for concept generation; moderate for SEO |

---

## F.5 The prompt registry

**Prompts are versioned files under source control, never inline strings.**

Each prompt directory contains: the prompt template, its output contract, its metadata (purpose, tier, temperature, token ceiling, cache policy, evaluation suite), and its evaluation cases.

| Rule | Reason |
|---|---|
| Semantic versioning | Output-shape changes are major; behavioural changes are minor; typo fixes are patches |
| Every generated entity records its prompt version | A prompt change never retroactively alters stored outputs |
| Prompts reviewed like code, with evaluation results attached | A prompt change is a behaviour change |
| Registry validated at process start | A missing contract or malformed template fails startup rather than failing in production |

### Standard prompt structure

```
SYSTEM
  Role and expertise framing
  Hard constraints — originality, no competitor reference, no protected terms
  The output contract restated in prose, alongside the machine-readable contract
  Policy for insufficient evidence: say so rather than inventing confidence

USER
  Task
  Grounding data          ← aggregated statistics only
  Constraints             ← excluded terms, anti-findings, resolved style
  <untrusted content>     ← external text, clearly delimited, appended last
  Output instruction
```

---

## F.6 Prompt-injection defence

Competitor listing text is attacker-influenceable in principle — anyone can publish a listing whose title contains instruction-like text. Registry responses and imported files are third-party content. All of it is treated as hostile.

| Control | Implementation |
|---|---|
| **Structural separation** | Untrusted content appears only inside delimited blocks, appended after all instructions, never interpolated into instruction text |
| **Explicit framing** | Every system prompt states that content in untrusted blocks is data to analyse and never an instruction |
| **Sanitisation** | Control characters stripped, delimiter-breaking sequences neutralised, per-field length caps, addresses removed where the task does not need them |
| **Least authority** | Prompts consuming untrusted data have **no tool access** and produce classification-only output with tightly constrained value sets |
| **Contract validation** | Values outside the allowed set are rejected, never coerced into the nearest match |
| **No echo to action** | Model output never constructs an address, query, path or provider parameter without validation against an allowed list |
| **Stage separation** | The stage that reads external text is a *different call* from the stage that generates. Generation receives only aggregated statistics. |

**The last control is the decisive one.** Because injected text has no path into the generative stage, the architecture makes injection structurally ineffective rather than merely filtered. An attacker who successfully injects instructions into a classification call can at most cause a misclassification — which the enum constraint bounds and which the statistics absorb.

---

## F.7 Cost engineering

### Levers, ordered by impact

| Lever | Approximate saving | Mechanism |
|---|---|---|
| **Aggregate before prompting** | ~85% | Contingency tables instead of raw listings. Four hundred listings as raw text is enormous; as aggregated statistics it is small. |
| **Content-hash vision caching** | 40–70% on repeat runs | Images analysed once ever, keyed by content and analyser version |
| **Prompt caching** | 30–50% on multi-call steps | Long shared context reused across sibling calls within a run |
| **Tier discipline** | 40–60% | The extraction tier handles the third of calls that are simple transforms |
| **Vision batching** | ~60% | Multiple images per call |
| **Concept-before-artwork gate** | ~75% of image spend | Only selected concepts render |
| **Deterministic-purpose response caching** | 10–20% | Identical input hash returns the stored answer |
| **Token ceilings** | 5–15% | Every call declares a realistic ceiling; contracts discourage rambling |

### Budget accounting

Cost is computed from a configured price table rather than from provider responses, so the figure is available immediately and consistently. It is reconciled against provider invoices monthly, with an alert on material variance.

Every call records its cost against the owning step, which records against the run, which records against the workspace monthly total. Cost is therefore attributable at every level without any additional instrumentation.

---

## F.8 Evaluation framework

**An AI feature without an evaluation is not shippable.**

| Evaluation type | Applied to | Gate |
|---|---|---|
| Contract conformance | Every prompt | 100% valid on first attempt across a fixed case set |
| Golden-set accuracy | Classification tasks | Macro-averaged accuracy thresholds per task, measured against hand-labelled examples |
| Groundedness | Concepts, gap explanations | **100%** — every cited finding must exist and the claim must match the stored statistic |
| Constraint compliance | SEO, concepts | **100%** after automatic repair |
| Safety recall | Legal screening | **Release gate** — high recall on cases that should reach blocked or high risk |
| Injection resistance | Any prompt consuming untrusted data | Zero instruction-following, zero out-of-vocabulary output |
| Distinctness | Concepts, SEO variations | Pairwise similarity below threshold |
| Rubric-judged quality | Concepts, SEO, briefs | Mean score threshold, with a human-calibrated sample |
| Cost regression | All | Mean tokens within tolerance of baseline |
| Latency regression | All | Percentile latency within tolerance of baseline |

### When evaluations run

| Trigger | Suite |
|---|---|
| Any change to a prompt or the AI service | Fast suite: contract, groundedness, constraints, injection, cost |
| Nightly | Full suite including golden sets and rubric judgement |
| Before any tier or model change | Full suite plus side-by-side human review of a sample |

### Production monitoring

A sample of production calls is scored asynchronously against the same rubrics. Contract-failure rate, repair rate and out-of-vocabulary rate are tracked per prompt version as leading indicators of degradation — these move before quality visibly drops.

---

## F.9 Multi-step patterns in use

| Pattern | Where | Why |
|---|---|---|
| Extract → validate → aggregate → generate | The whole pipeline | Keeps generation grounded in verified data |
| Parallel independent generation | Success-derived and gap-derived concepts | Different grounding, no cross-contamination, halved latency |
| Generate → embed → deduplicate → refill | Concepts | Guarantees twenty genuinely distinct ideas rather than twenty filled slots |
| Classify → rule table → decide | Legal screening | The model judges; a deterministic table decides |
| Brief → render → validate → repair | Artwork | Separates creative specification from mechanical validation |
| Generate along declared axes | SEO | Forces genuine diversity rather than ten paraphrases |
| Self-critique before output | Artwork briefs | The prompt requires checking the brief against the print-constraint list before emitting |

**Deliberately not used: autonomous agent loops with tool access.** Every step here has a known shape, a known cost and a known output contract. Agentic freedom would break the cost ceiling, the reproducibility guarantee and the injection defence, while buying nothing.

---

## F.10 AI failure handling

| Failure | Behaviour |
|---|---|
| Contract invalid after repair and retry | Step fails explicitly; raw output preserved and viewable; retry available |
| Model declines to respond | Recorded. For legal-adjacent prompts this **escalates the risk level** rather than being retried around. |
| Timeout | Retried at the same tier, then escalated one tier, then failed |
| Rate limited | Backoff honouring any provider hint; the step remains running with progress messaging |
| Provider outage | Circuit breaker opens; dependent steps move to blocked-external; the run remains resumable and resumes automatically |
| Cost anomaly | A single call materially above expectation raises an alert but is allowed to complete — killing it mid-flight wastes the spend already incurred |
| Degenerate output | Detected by contract validators; counted as a contract failure |

---

## F.11 Data governance for AI

| Rule | Detail |
|---|---|
| No training on our data | Confirmed contractually and recorded in the subprocessor register |
| Minimum necessary context | Prompts receive aggregates, not raw records. No credentials, no personal data, no buyer data ever enters a prompt. |
| Raw response retention | Ninety days, then purged. Structured outputs are retained indefinitely as domain data. |
| Reproducibility | Prompt version, model identifier, input hash, temperature and seed are sufficient to explain any historical output |
| Attribution | Every model-produced artefact is identifiable as such in the interface, with its prompt version accessible |
| Human override | Every output is editable, and edits are recorded — so evaluation data is not polluted by human corrections |

---

# Part G — Integration Architecture

## G.1 The adapter contract

Every external dependency sits behind an adapter with an identical internal shape and an identical set of responsibilities.

| Responsibility | Requirement |
|---|---|
| **Schema validation** | Every response parsed and validated at the boundary. Unknown fields tolerated; missing required fields are errors. Internal code never sees an unvalidated shape. |
| **Retry** | Idempotent operations retry with exponential backoff and jitter. Non-idempotent operations never retry automatically without an idempotency key. |
| **Rate limiting** | Distributed token bucket keyed by provider and workspace, shared across all processes |
| **Circuit breaking** | Opens on consecutive failures or a sustained error rate; half-open probe after backoff |
| **Timeouts** | Connect and read timeouts, plus a total call budget enforced by cancellation |
| **Cost recording** | Every call reports cost to the budget guard and writes a ledger entry |
| **Error translation** | Provider errors mapped to the internal error catalogue with specific remedies. Raw text is logged, never surfaced. |
| **Credential handling** | Fetched per call from the credential service; never held in module scope |
| **Instrumentation** | Every call produces a trace span and a ledger row |
| **Testing** | Fixture-backed. **No test in the build pipeline ever calls a live external service.** |

**Enforced by automated analysis:** no vendor library may be imported outside its own adapter directory. A violation fails the build.

---

## G.2 Market data integration — the hardest problem

### The honest position

**The dominant third-party research tool in this market does not publish a general-purpose public interface.** Any architecture that depends on programmatic access to it is building on ground that can shift without notice.

This is not a reason to avoid using it — its sales and revenue estimates are the highest-value single input to the product, and the operator already subscribes. It **is** a reason to build the integration as a chain of interchangeable adapters with a path that cannot break.

### The governing commitment

> **The product must remain fully functional — reduced in fidelity, not in capability — using only public marketplace data and files the operator supplies themselves.**

Everything above that baseline is an enhancement, and every enhancement is gated by capability detection.

### The adapter chain

```
        Engine requests shops or listings
                    │
                    ▼
        ┌───────────────────────────┐
        │   MARKET DATA SERVICE     │
        │   probes available        │
        │   adapters, resolves the  │
        │   chain, merges by field  │
        └───────────┬───────────────┘
                    │
     ┌──────────────┼──────────────┬───────────────┬──────────────┐
     ▼              ▼              ▼               ▼              ▼
┌─────────┐  ┌────────────┐  ┌──────────┐  ┌────────────┐  ┌──────────┐
│PROVIDER │  │  OPERATOR  │  │ ASSISTED │  │   PUBLIC   │  │ FIXTURE  │
│   API   │  │    FILE    │  │  SESSION │  │MARKETPLACE │  │(testing) │
│         │  │  (DEFAULT) │  │(OPT-IN)  │  │    API     │  │          │
│ green   │  │   green    │  │  amber   │  │   green    │  │   n/a    │
└─────────┘  └────────────┘  └──────────┘  └────────────┘  └──────────┘
                    │              │              │
                    └──────────────┴──────────────┘
                                   │
                    field-level merge by confidence,
                    with provenance recorded per field
```

### Adapter A — Operator file import (default, always available)

The operator exports data from the tool they already subscribe to, using that tool's own export feature, and uploads the file. **No automated access to anyone's systems occurs.**

| Aspect | Detail |
|---|---|
| Trigger | Manual upload or drag-and-drop in settings |
| Format handling | Header signature detection with fuzzy column mapping; the operator confirms once per format and it is remembered |
| Validation | Per-row validation with downloadable rejection reasons |
| Freshness | Export date where present, upload time otherwise; the interface warns when data ages |
| Confidence | Medium for estimates, high for directly observed fields |
| Capabilities | Shop and listing sales and revenue estimates, review counts, keyword volume — everything that matters |
| **Failure modes** | **None that any external party can cause** |

**This being the default is what makes the product's core value independent of a scraping arms race.**

### Adapter B — Provider interface (if one exists)

If a documented interface exists now or later — including through a partner or affiliate arrangement — this is a thin client following the standard adapter contract. It is written to be trivially added; it is not assumed to exist.

**Recommended commercial action:** approach the provider for interface or partner access before implementation begins, and record the outcome. A formal arrangement converts the highest-value input from conditional to guaranteed and is worth pursuing on commercial grounds alone.

### Adapter C — Assisted session (optional, off by default)

Operator-authorised, operator-credentialed retrieval of data the operator is already entitled to see.

| Guardrail | Rule |
|---|---|
| Default | **Disabled.** Requires explicit opt-in with a plain-language consent screen naming the risk. |
| Credentials | The operator's own only |
| Rate | Roughly one request per three seconds, low daily ceiling, randomised jitter, no parallelism |
| Response to refusal | Halt immediately; open a circuit breaker for a full day |
| Scope | Only data the authenticated operator can already see |
| Transparency | Every request logged; a live counter shown in settings |
| Kill switch | One toggle; the system falls back without interruption |
| **Detection evasion** | **Never implemented.** No fingerprint spoofing, no challenge solving, no address rotation. |

**Enforced architecturally, not by policy alone:** the adapter interface provides no capability for evasion, and a build-time check verifies no such logic exists anywhere in the codebase. If access requires evasion, the adapter reports itself unavailable — which is the correct engineering answer, not a limitation.

### Adapter D — Public marketplace interface (always available)

Provides **measured** data — real prices, real listing metadata, real review counts, real shop ages — but **not** sales or revenue estimates.

| Data | Available | Confidence |
|---|---|---|
| Listing title, description, tags, price, images, category | Yes | High |
| Listing creation date, state, favourites | Yes | High |
| Shop name, opening date, review count, rating, total sales | Yes | High |
| Per-listing sales estimate | **No** | — |
| Per-listing revenue estimate | **No** | — |
| Keyword search volume | **No** | — |

**Derived proxies when this is the only source**, all clearly labelled with the method named:

| Missing metric | Proxy |
|---|---|
| Listing sales | Review count multiplied by a niche-calibrated review-to-sale ratio, scaled by listing age |
| Listing revenue | Proxy sales multiplied by price |
| Review velocity | Reviews accumulated between snapshots — which is why the snapshot design matters |
| Trend | Change in review velocity across snapshots |

This is materially weaker than estimate data, and the product says so plainly. **But the genuinely differentiated analysis — visual style, layout, pricing structure, image counts, listing construction, SEO — depends only on data the public interface provides.** The product remains valuable in this mode.

### Capability-driven degradation

The pipeline reads available capabilities at run start and adapts.

| Missing capability | Effect | What the operator sees |
|---|---|---|
| Sales estimates | Cohorts built from proxies; confidence downgraded; evidence thresholds tightened | A banner naming the limitation and offering one-click file import |
| Revenue estimates | Shop selection uses sales, reviews and velocity only | Selection rationale shows which criteria were used |
| Keyword volume | Keyword ranking uses competitor-tag weighting only | The keyword evidence panel names the method |
| Trend series | Trend uses review-velocity change | The sub-score is badged with its fallback method |
| All estimate capabilities | The run is marked degraded throughout | A prominent banner with the specific remedy |

**The run never fails because a provider is missing.** It produces a smaller, honestly labelled result.

### Field-level merge

Records from multiple sources describing the same entity are merged **field by field**, never row-replaced:

1. Measured values beat estimated values.
2. Higher confidence beats lower.
3. Fresher beats staler.
4. Ties broken by configured provider priority.

The winning source is recorded per field. This is what allows the interface to say honestly: *"price: measured from the marketplace; monthly sales: estimated, medium confidence, from your export of three days ago."*

Conflicting estimates are both retained. The merge selects one, and the disagreement is visible in the listing detail view rather than hidden.

### Compliance posture

| Adapter | Tier | Consent | Default |
|---|---|---|---|
| Operator file import | Green | None needed | **Enabled** |
| Provider interface | Green | None needed | If configured |
| Public marketplace | Green | None needed | **Enabled** |
| Assisted session | **Amber** | **Explicit, informed** | **Disabled** |
| Anything requiring evasion | **Red** | — | **Never built** |

**Standing rules:** provider terms are reviewed at every phase gate with the date recorded and displayed in settings. Only public commercial data is collected — no personal data of buyers, reviewers or shop owners. Competitor images are stored transiently for analysis, retained for a bounded period, **never used as generative input**, and never redistributed. A single toggle disables all amber adapters instantly.

---

## G.3 AI provider integration

| Concern | Approach |
|---|---|
| Abstraction | Capability tiers, not model names. Binding lives in configuration. |
| Output contract | Every call bound to a schema, with validation, one repair and one stricter retry |
| Caching | Deterministic purposes cached by input hash; creative purposes never |
| Prompt caching | Long shared context marked for reuse across sibling calls in a run |
| Cost | Computed from a configured price table for immediacy; reconciled against invoices monthly |
| Rate limiting | Per-tier concurrency and token-rate budgets, distributed |
| Failure | Circuit breaker per tier; blocked-external state rather than run failure |
| Portability | A second provider could be added behind the same tier interface without touching any engine |

Detail in [Part F](#part-f--ai-architecture).

---

## G.4 Image generation integration

| Concern | Approach |
|---|---|
| Abstraction | A provider interface declaring capabilities: transparent background, negative prompting, seed control, style presets, aspect ratios, maximum resolution |
| Prompt compilation | Performed by the adapter, not the engine — provider-specific tuning stays in one file |
| Provider-side rewriting | **Disabled.** It would break reproducibility, defeat the negative constraints and invalidate evaluations. |
| Variants | Generated as independent calls so one failure costs one variant |
| Seeds | Deterministic per brief and variant index, enabling reproduction and controlled iteration |
| Concurrency | Deliberately low, bounding the blast radius of a runaway loop |
| **Input constraint** | **No code path accepts an external image as input**, other than the artwork's own prior renditions for upscaling |

**Portability in practice.** Only generation is provider-specific. Background removal, edge refinement, cropping, upscaling, validation, rendition production, embedding, originality checking, vectorisation and safety review are all provider-independent. Substituting a provider means a new adapter, six style-template mappings and an evaluation run.

---

## G.5 Marketplace integration

### Authorisation

Authorisation-code flow with proof-key exchange. The verifier is stored server-side against an opaque state value with a short lifetime. State is verified on callback; any mismatch aborts before any token exchange.

| Concern | Approach |
|---|---|
| Scope | **Minimum necessary only.** Listing read and write, shop read, transaction read. Nothing touching personal data, messaging, or billing. |
| Storage | Encrypted; never returned to the client; masked representation served separately |
| Refresh | Proactive, before expiry, with a distributed lock preventing concurrent refresh from invalidating the token |
| Rotation | New credentials written in the same transaction that supersedes the old |
| Expiry warning | Countdown surfaced in the interface well before the deadline |
| Revocation | Moves pending operations to a resumable needs-reauthorisation state — **paused, never failed and discarded** |

### Rate management

The marketplace enforces both a per-second and a daily limit.

| Control | Approach |
|---|---|
| Token bucket | Distributed, shared across all processes |
| **Reserved allocation** | A portion of the daily budget is held back exclusively for publishing and performance synchronisation |
| Priority classes | Publishing preempts synchronisation, which preempts research |
| Conditional requests | Used where supported, so unchanged data costs a cheap response rather than a full payload |
| Backoff | Honours provider hints; repeated limiting opens the breaker |

**The reserved allocation matters.** Without it, a heavy research day could consume the daily quota and leave the operator unable to publish — a failure mode that would be discovered at exactly the wrong moment.

### Publishing decomposition

Eight independently idempotent operations, each with its own key, status and retry policy:

```
upload artwork → create product → fetch mockups → create draft listing
   → upload images → set inventory → link fulfilment → publish
```

**Duplicate prevention at four layers:** request-level idempotency keys with response replay; application-level pre-flight checks for an existing listing identifier; reconciliation that adopts an existing listing on ambiguous timeout rather than creating a second; and a database uniqueness constraint as the final backstop.

### Draft-only automation

**There is no code path that creates an active listing.** Creation sets draft state; activation is a separate operation with a full server-side pre-flight re-evaluating every hard check from current data.

### Policy compliance

| Area | Handling |
|---|---|
| Production method declaration | Set correctly on every listing |
| Production partner | Declared; its presence is a blocking pre-publish check |
| Intellectual property | Screened before generation, after generation, and before publishing |
| Restricted terms | Checked in listing content generation and again before publishing |
| Content provenance | Full generation history retained so any disclosure requirement can be met |
| Prohibited automation | The system never messages buyers, never influences reviews, never interacts with competitor listings, and never publishes without explicit human confirmation |

---

## G.6 Fulfilment integration

### Catalogue synchronisation

| Data | Frequency | Reason |
|---|---|---|
| Product structures | Weekly | Changes slowly |
| Provider list | Weekly | Changes slowly |
| **Variant costs** | **Daily** | **A stale cost silently corrupts every margin figure** |
| Shipping costs | Weekly | |

**Cost drift monitoring is a first-class feature, not housekeeping.** When a variant's cost moves beyond a threshold, unpublished drafts using it are repriced automatically and the operator is alerted. Without this, margin erodes invisibly across an entire portfolio.

**Scope control:** only products mapped to supported categories and whitelisted providers are synchronised in full, keeping the operation to hundreds of calls rather than tens of thousands.

**Provider reliability rating** is maintained locally from observed production times and problem rates, and feeds recommendation ranking. A cheap provider with poor reliability is not actually cheap.

### Product creation

| Concern | Approach |
|---|---|
| Artwork upload | Deduplicated by content hash — the same artwork is never uploaded twice, even across drafts |
| Placement | Computed deterministically per product type, adjustable by the operator, persisted and reapplied on updates |
| Mockups | Retrieved asynchronously with backoff or webhook; **never blocking a request thread** |
| Mockup selection | Filtered to a preferred set rather than importing everything; ordered by what performs in the niche |
| Updates | Modify the existing product; **never create a second** |
| Reconciliation | On ambiguous timeout, search recent products for a matching signature and adopt |

### The publishing bridge decision

Two routes exist. The default is deliberate.

| | **Direct marketplace control (default)** | Fulfilment-provider bridge |
|---|---|---|
| Listing content control | **Full** — our title, our thirteen tags, our structured description, our category, our image order | Limited — provider mapping governs |
| Draft-only guarantee | **Yes, we control state** | Depends on provider behaviour |
| Failure isolation | Per operation | Opaque |
| **Verdict** | **Default.** Listing content is the product's core value; ceding control of it is unacceptable. | Available as a fallback when marketplace access is unavailable. |

In the default mode, after the marketplace listing exists, the fulfilment product must be linked to it so orders route correctly. **This step is mandatory and appears as a blocking pre-publish check**, because omitting it produces a silent failure in which the listing sells and nothing is manufactured.

---

## G.7 Trademark registry integration

| Concern | Approach |
|---|---|
| Coverage | Multiple jurisdictions, restricted to the goods classes relevant to the supported products |
| Matching | Exact, fuzzy and phonetic |
| Caching | Thirty days by normalised term — registrations do not change hourly, and uncached lookups would make screening slow and expensive |
| Parallelism | Jurisdictions queried concurrently |
| Failure | A registry outage degrades screening with an explicit warning; it does not silently pass concepts through |
| Retention | Raw responses retained for seven years to support any future dispute |

**A registry failure must never appear as a clean result.** If a jurisdiction could not be checked, the screening records that fact and the risk assessment reflects reduced coverage.

---

## G.8 Storage integration

| Concern | Approach |
|---|---|
| Interface | Standard object-storage protocol only, so the provider is interchangeable |
| Upload | Direct from browser to storage via a signed target — bytes never pass through the application |
| Download | Direct from storage via a signed link |
| Keys | Generated server-side from identifiers. **User input never contributes to a storage path.** |
| Deduplication | By content hash before any write |
| Lifecycle | Retention tiers enforced by storage rules, not application code |
| Versioning | Enabled, so accidental deletion is recoverable |

---

## G.9 Integration failure matrix

| Provider unavailable | Effect | Operator experience |
|---|---|---|
| Market data (all adapters) | Research continues on public marketplace data | Degraded banner with the specific remedy |
| AI provider | Dependent steps become blocked-external | Run pauses, resumes automatically when the breaker closes |
| Image generation | Artwork step blocked; concepts stay queued | Nothing else is affected; retry available with breaker status shown |
| Marketplace (read) | Competitor collection degrades | Warning; existing data still analysed |
| Marketplace (write) | Publishing pauses | Clear reconnection or retry prompt; nothing lost |
| Fulfilment | Product building pauses | Draft retains all state; targeted retry available |
| Trademark registry | Screening degrades with reduced coverage | **Explicit warning — never a silent clean result** |
| Object storage | Asset operations fail | Steps requiring assets pause; metadata unaffected |

**The universal rule:** a provider failure pauses work and preserves state. It never destroys work, and it never silently produces a result that looks complete.

---

# Part H — Background Job System

## H.1 What runs asynchronously, and why

Anything that takes longer than a request should, costs money, calls an external service, or must survive a restart.

| Work | Duration | Why asynchronous |
|---|---|---|
| Research run steps | 30 s – 15 min each | Far exceeds any request timeout |
| Competitor collection | 3–5 min | Dozens of external calls |
| Visual style analysis | 2–5 min | Batched model calls across hundreds of images |
| Concept generation | 60–90 s | Model latency |
| Legal screening | 10–30 s per concept | Parallel registry lookups |
| Artwork generation | 2–4 min per concept | Provider latency plus multi-stage processing |
| Artwork post-processing | 30–90 s | CPU-intensive image work |
| SEO generation | 60 s | Model latency |
| Fulfilment product creation | 1–2 min | Includes waiting for mockups |
| Marketplace publishing | 30–60 s | Multiple sequential operations |
| Performance synchronisation | Continuous | Scheduled, high volume |
| Catalogue synchronisation | 5–20 min | Hundreds of calls |
| Learning recalibration | 1–5 min | Computation over the full outcome set |
| Retention and consistency | Minutes | Scheduled maintenance |
| Notification delivery | Seconds | Should never block anything |

**The governing rule: no request handler ever waits for an external service to finish work.** It enqueues and returns.

---

## H.2 Queue topology

Separate queues rather than one, because isolation is the cheapest form of resilience.

| Queue | Concurrency | Contents | Why isolated |
|---|---|---|---|
| `research` | Moderate | Run start, analytical steps | CPU-bound; must not be starved |
| `vision` | Higher | Image analysis batches, embeddings | Latency-bound, benefits from parallelism |
| `artwork` | **Low** | Generation, background removal, upscaling, vectorisation | Expensive — low concurrency bounds runaway spend |
| `publish` | **Low** | Fulfilment and marketplace operations | Strictly rate-limited upstream; high concurrency would cause rate-limit storms |
| `sync` | Moderate | Performance sync, catalogue sync, listing resync | Background priority; must never delay interactive work |
| `maintenance` | Low | Retention, consistency, reaping, partitions, search reindex | Off-peak |
| `notify` | Moderate | Email, webhook delivery | Fast and independent; must never be blocked behind slow work |

**The reasoning is specific:** an artwork backlog must never delay a publish. A slow registry lookup must never starve analysis. A catalogue synchronisation must never consume the rate-limit budget that publishing needs. Separate queues with separate concurrency limits make each of these structurally impossible rather than merely unlikely.

---

## H.3 Job payload discipline

**Payloads carry identifiers, never entities.**

A payload contains: workspace identifier, run identifier, step key, entity identifier, attempt number, correlation identifier. The handler re-reads current state from the database.

**Why this matters more than it appears.** Queued jobs outlive deployments. If a payload embedded a serialised entity, a schema change would break every job already in flight. Identifiers are stable across every deployment. This single discipline prevents an entire class of deployment-time failures.

---

## H.4 Job states

```
                    ┌──────────┐
                    │ WAITING  │  in queue, not yet claimed
                    └────┬─────┘
                         │ worker claims
                    ┌────▼─────┐
                    │  ACTIVE  │  executing, heartbeating
                    └────┬─────┘
             ┌───────────┼───────────┬──────────────┐
             │           │           │              │
        ┌────▼────┐ ┌────▼────┐ ┌───▼──────┐ ┌────▼──────┐
        │COMPLETED│ │ FAILED  │ │ DELAYED  │ │ CANCELLED │
        └─────────┘ └────┬────┘ └───┬──────┘ └───────────┘
                         │          │ backoff elapsed
                         │          └──────► WAITING
                         │ attempts remain
                         └──────────────────► DELAYED
                         │ attempts exhausted
                         ▼
                    ┌──────────┐
                    │DEAD LETTER│  full context preserved, alert raised
                    └──────────┘
```

### Step states — the durable layer

Job states are transient and live in the queue. **Step states are durable and live in the database.** They are related but distinct:

| Step state | Meaning |
|---|---|
| `pending` | Not started, or waiting on dependencies |
| `running` | Claimed by a worker, heartbeating |
| `succeeded` | Output persisted transactionally with the status |
| `failed` | Attempts exhausted, error recorded |
| `skipped` | Optional step deliberately bypassed, with the degradation recorded |
| `blocked_external` | A provider circuit breaker is open — **not a failure**, will resume automatically |
| `cancelled` | Operator cancelled |

**Why two layers.** If the queue were lost entirely, every incomplete run could be reconstructed and re-enqueued from step states in the database. The queue is treated as a delivery mechanism, not a source of truth — which is what makes it safe to run on managed infrastructure with no durability guarantee.

---

## H.5 The orchestration loop

```
1.  Claim the next step, skipping any row another worker holds
2.  Verify every dependency has succeeded
3.  Check the cancellation flag
4.  Reserve the estimated cost against the run budget
        └── if denied → set run to budget-paused, publish event, stop
5.  Mark the step running, increment the attempt, start heartbeating
6.  Publish a step-started event
7.  Execute the handler
        ├── check cancellation before every external call
        └── update the heartbeat periodically for long steps
8.  In ONE transaction:
        ├── persist the output
        ├── set the step to succeeded
        ├── record actual cost against the step and the run
        └── record token usage
9.  Publish a step-completed event
10. Enqueue every step whose dependencies are now satisfied
```

**Step eight is the critical guarantee.** Output, status and cost commit together. There is no window in which a step is marked succeeded without its output, or in which cost is lost or double-counted.

---

## H.6 Error handling

### Classification determines behaviour

| Class | Examples | Behaviour |
|---|---|---|
| **Transient** | Network timeout, provider 5xx, connection reset | Retry with exponential backoff and jitter |
| **Rate limited** | Provider quota | Retry honouring the provider's hint; step remains running with progress messaging |
| **Circuit open** | Repeated provider failure | Step moves to blocked-external; **not a failure**; resumes automatically |
| **Validation** | Provider rejected our request | **No retry** — retrying identical invalid input wastes money and time. Fail with a specific remedy. |
| **Contract** | Model output failed its schema | One repair, one stricter retry, then fail with the raw output preserved |
| **Budget** | Reservation denied | Pause the run, notify, await an explicit decision |
| **Authorisation** | Credentials revoked or expired | Move to needs-reauthorisation; **paused and resumable, never failed** |
| **Partial** | Some items in a batch failed | **Succeed with the successful portion**, record what failed and the count |
| **Fatal** | Programming error, corrupted state | Fail immediately, alert, dead-letter with full context |

### Partial success is a first-class outcome

This deserves emphasis because it is where most pipelines get it wrong.

- One shop failing must not fail competitor collection — nine shops of data is far better than none.
- Some images failing analysis must not fail style extraction — but **the exclusion count must be reported**, because presenting conclusions drawn from 91% of listings as though they covered all of them would be dishonest.
- Some images failing to upload must not fail publishing — the remaining images are usable and a targeted retry repairs the gap.

**The rule:** partial results are returned with an explicit statement of what is missing. Silently succeeding with incomplete data is treated as a defect.

---

## H.7 Retry policy

| Job class | Attempts | Backoff | Notes |
|---|---|---|---|
| Analytical steps | 3 | Exponential from seconds | Deterministic; retry is usually safe |
| External reads | 3 | Exponential with jitter | Idempotent |
| Model calls | 2 plus contract repair | Exponential | Cost-bounded |
| Artwork generation | 2 automatic | Fixed | Then requires operator confirmation — each attempt costs real money |
| Publishing operations | 3 | Exponential | Protected by idempotency keys |
| Performance sync | 5 | Long exponential | Low urgency, high tolerance |
| Notifications | 5 | Long exponential | Independent of everything |

### Jitter is mandatory

Without randomised backoff, a provider outage that fails fifty jobs simultaneously causes all fifty to retry simultaneously, guaranteeing another synchronised failure. Jitter spreads retries and is the difference between recovery and a thundering herd.

### Retry budget

Retries count against the run's cost budget. A step that has consumed a disproportionate share of the budget in retries stops and asks, rather than continuing to spend.

---

## H.8 Crash recovery

**The most important reliability behaviour in the system.**

```
Worker dies mid-step
        │
        ▼
Step remains 'running' with a heartbeat that stops advancing
        │
        ▼
Maintenance task runs every minute:
   find steps that are 'running' with a stale heartbeat and no live worker lease
        │
        ▼
Mark them failed with reason 'worker lost'
        │
        ▼
Attempts remaining? ──yes──► requeue at the next attempt
                    └──no───► fail the step, notify, allow manual retry
```

**Why heartbeats rather than timeouts.** Steps have wildly different durations — some take seconds, some take fifteen minutes. A fixed timeout would either kill legitimate long steps or leave dead workers undetected for far too long. A heartbeat detects death in about a minute regardless of expected duration.

**Cost reservations self-release.** Reservations carry an expiry matching the step timeout, so a crashed step does not permanently hold budget.

**Verification:** a chaos test kills a worker mid-run and asserts that the run produces identical final output with **zero duplicate external effects**. This is not an optional test.

---

## H.9 Cancellation

| Aspect | Behaviour |
|---|---|
| Signal | A flag set in the shared store, checked by the orchestrator and by handlers |
| Check points | Before every external call, and between batch items |
| Latency | Effective within ten seconds |
| Completed work | **Preserved.** Cancellation is not deletion. |
| In-flight external calls | Allowed to complete if already dispatched — the money is already spent; discarding the result wastes it |
| Cost | Recorded up to the cancellation point |

---

## H.10 Scheduled work

A single scheduler process holds a distributed lock. **Every scheduled job is idempotent**, so a missed tick or a duplicated tick during failover is harmless.

| Job | Frequency | Purpose |
|---|---|---|
| Performance synchronisation | Daily, staggered; more frequent for recently published listings | Fetch views, favourites, orders, revenue and state |
| Catalogue structure sync | Weekly | Product structures and providers |
| **Variant cost sync** | **Daily** | **Detect cost drift before it silently erodes margin** |
| Token refresh | Hourly | Refresh credentials before expiry |
| Step reaper | Every minute | Detect and requeue steps whose worker died |
| Retention | Nightly | Purge expired data per policy |
| Partition management | Weekly | Create upcoming partitions, archive expired ones |
| Consistency check | Nightly | Verify lineage integrity, asset reachability, status coherence |
| Percentile recomputation | Nightly | Refresh age-adjusted performance rankings |
| Outcome record build | Daily | Assemble learning-loop training data |
| Recalibration evaluation | Weekly | Assess whether a proposal is warranted |
| Cost reconciliation | Monthly | Compare recorded costs against provider invoices |
| Backup verification | Monthly | Restore drill into a scratch environment |
| Search reindex | Nightly | Refresh the search index |

**Why the scheduler is a singleton.** Duplicate cron execution is a real risk with multiple instances. A single process holding a lock, with automatic failover, is simpler and far more debuggable than distributed leader election.

---

## H.11 Dead letter handling

A job exhausting its attempts moves to a dead-letter record with: the full payload, every attempt's error, the correlation identifier, and enough context to diagnose without reproduction.

| Behaviour | Detail |
|---|---|
| Alerting | Every dead-letter entry raises an alert |
| Queue protection | A poison message never blocks its queue |
| Inspection | Dead-lettered jobs are viewable with full context |
| Replay | Individually replayable after a fix |
| Resolution | Marked resolved with a note, forming a defect record |

---

## H.12 Observability of background work

| Metric | Purpose |
|---|---|
| Queue depth per queue | Backlog size |
| **Queue age (oldest waiting job)** | **The primary scaling signal** — depth without age is meaningless |
| Processing time distribution per job type | Performance regression detection |
| Failure rate per job type | Reliability |
| Retry rate | Leading indicator of provider trouble |
| Dead letter rate | Defect signal |
| Active workers per queue | Capacity |
| Cost per job type | Economic attribution |
| Step duration against expectation | Detects steps drifting slower |

**Alerting thresholds:** queue age beyond several minutes, dead-letter entries, run failure rate above a small percentage, circuit breaker open for a sustained period, and any publishing failure.

**Tracing:** the trace context propagates through the job payload, so a run's entire execution — across the web tier, the queue, multiple workers and every external call — is viewable as a single connected trace.

---

## H.13 Capacity and scaling

| Signal | Action |
|---|---|
| Queue age rising on `research` | Add workers |
| Queue age rising on `vision` | Add workers; check for cache-hit-rate degradation |
| Queue age rising on `artwork` | **Do not simply add workers** — check whether this is a runaway retry loop or a cost problem first |
| Queue age rising on `publish` | **Do not add workers** — this is almost always upstream rate limiting, and more workers would make it worse |
| Dead letters accumulating | Investigate before scaling; scaling a broken job type multiplies the failure |

**The distinction between `artwork`/`publish` and everything else is deliberate.** For most queues, backlog means insufficient capacity. For these two, backlog usually means something is wrong upstream, and adding capacity makes it worse. Encoding that in the runbook prevents the instinctive and incorrect response.

---

# Part I — Security Architecture

## I.1 What we are actually protecting

Single-user does not mean low-stakes. Ranked by consequence of compromise:

| Rank | Asset | Consequence if compromised |
|---|---|---|
| **1** | Marketplace credentials with write access | An attacker can create, alter or delete listings on a live revenue-generating shop. Reputational and financial damage, potentially unrecoverable. |
| **2** | Fulfilment provider credentials | Product creation and potentially production charges |
| **3** | AI and image generation keys | Direct financial loss through spend |
| **4** | The proprietary outcome dataset | Loss of the product's only durable competitive asset |
| **5** | Operator account credentials | Full system access |
| **6** | Competitor market data | Commercially sensitive, low absolute severity |

**The design is built to a multi-tenant standard from the start**, because retrofitting security is harder than retrofitting features, and because assets one and two are already live from the first week of use.

---

## I.2 API key and credential protection

### Envelope encryption

```
   Key management service holds the master key — it never leaves
              │
              │ wraps
              ▼
   Per-workspace data key — stored wrapped, alongside the ciphertext
              │
              │ encrypts
              ▼
   Credential bundle — authenticated encryption, bound to its context
```

| Control | Detail |
|---|---|
| **Decryption location** | Worker and API process memory only, on demand |
| **In-memory lifetime** | Seconds, then discarded. Cleared immediately on any integration change. |
| **Never** | Written to logs, traces, error messages, client responses, environment dumps or backups in plaintext |
| **Client display** | A masked representation only, served from a separate endpoint that performs no decryption |
| **Context binding** | The encryption is bound to the workspace, provider and key version, so a ciphertext cannot be replayed into another workspace's row |
| **Rotation** | A version field supports re-wrapping under a new master key without re-authenticating any provider |
| **Table separation** | Ciphertext lives in its own table with tighter database grants, so routine health-check queries never touch encrypted material |

### Application secrets

| Control | Detail |
|---|---|
| Source | Platform secret store, injected at runtime. Never in source, never baked into images. |
| Validation | Checked at process start. **The process refuses to start on a missing or malformed secret** rather than failing at first use. |
| Scanning | Automated secret detection on every change and periodically across full history. A hit blocks the build. |
| Rotation | A documented procedure per secret, with its blast radius and rollback recorded |
| Least privilege | Separate database credentials per service. The web role cannot perform destructive schema operations; the worker role cannot read the audit log. |

### Provider scope minimisation

Only the minimum necessary permissions are requested from each provider, and the requested set is shown to the operator before they authorise. Permissions touching personal data, messaging, favourites or billing are never requested.

---

## I.3 User authentication

| Control | Requirement |
|---|---|
| Password storage | Memory-hard hashing with substantial memory cost and versioned parameters supporting upgrade on next login |
| Password policy | Minimum length enforced; checked against known-breached passwords using a privacy-preserving range query; no arbitrary composition rules |
| **Second factor** | **Mandatory.** Time-based codes with a narrow drift window. The secret is encrypted at rest. |
| Recovery | Single-use codes, displayed exactly once, stored hashed |
| Brute force | Per-account exponential lockout and per-address sliding-window limiting |
| Enumeration | Identical responses and normalised timing whether or not an account exists |
| Session tokens | Opaque random values, hashed at rest, never self-contained |
| Cookies | Marked secure, HTTP-only and same-site |
| Rotation | On login and on any privilege change |
| Expiry | Idle expiry of days, absolute expiry of weeks |
| Revocation | Individual and bulk session revocation; automatic on password change |

**Why opaque tokens rather than self-contained ones.** A self-contained token remains valid until it expires, regardless of what happens server-side. Given that this session controls write access to a live shop, the ability to revoke instantly is worth the lookup cost.

### Step-up authentication

Recent re-authentication is required for: changing credentials, connecting or disconnecting an integration, overriding a high legal risk, raising a spending budget, activating a scoring configuration change, and **publishing**.

**The principle:** anything irreversible, anything that spends significant money, and anything that accepts legal risk requires proof that the person at the keyboard is still the account holder.

---

## I.4 Authorisation

**Phase 1 has one user with the owner role. The mechanism is nonetheless complete**, because enabling it later must not require touching every handler.

### The single most important rule

**Workspace scope is derived from the session, server-side, and is never accepted from client input.** Every entity access re-resolves the identifier against the workspace, and a mismatch returns *not found* — never *forbidden*. The API must never confirm that another workspace's data exists.

### Permission matrix — defined now, enforced from the multi-user phase

| Capability | Owner | Admin | Member | Viewer |
|---|---|---|---|---|
| View reports, concepts, artwork | Yes | Yes | Yes | Yes |
| Start a research run (spends money) | Yes | Yes | Yes | No |
| Generate artwork (spends money) | Yes | Yes | Yes | No |
| Override a legal risk | Yes | Yes | No | No |
| Publish to the marketplace | Yes | Yes | No | No |
| Manage integrations and credentials | Yes | Yes | No | No |
| Change budgets and fee model | Yes | No | No | No |
| Activate a scoring configuration | Yes | Yes | No | No |
| Manage members and billing | Yes | No | No | No |

**Note what this matrix already provides:** a `member` who can research and design but cannot publish or accept legal risk is exactly the delegation model a team needs. The approval gate built for safety turns out to be the primitive for delegation.

### Defence in depth

Authorisation is enforced at the application layer **and** at the database layer. Row-level isolation policies are written and tested from day one; production switches to an isolation-enforcing database role at the multi-user phase. That switch is a deployment change, not a code change.

---

## I.5 Data protection

| Layer | Control |
|---|---|
| **In transit** | Modern transport encryption everywhere, including application-to-database. Strict transport security with preloading. |
| **At rest** | Full-disk encryption on managed database and object storage; additional column-level envelope encryption for credentials and second-factor secrets |
| **Object storage** | Private buckets, no public access, no unsigned path to any object, server-side encryption, versioning enabled |
| **Backups** | Encrypted, access-controlled, and **restore-tested monthly** |
| **Deletion** | Soft delete with a recovery window, then hard purge including stored objects. Export offered before deletion. |
| **Minimisation** | Only public commercial data collected. **No buyer data, no reviewer identities, no personal data of competitor shop owners beyond a public shop name.** |
| **AI boundary** | No credentials, no personal data and no raw records enter any prompt — only aggregated statistics |
| **Training** | Contractual confirmation that data is not used to train third-party models, recorded in the subprocessor register |

### Retention

| Data | Period |
|---|---|
| Raw competitor observations | Bounded, then archived; derived analysis retained indefinitely |
| Provider request and response bodies | Short — invaluable for days, worthless thereafter |
| Raw AI responses | Bounded; structured outputs retained as domain data |
| Progress events | Short |
| Owned entities — runs, concepts, artwork, listings | Indefinite until deletion |
| **Legal screenings and registry responses** | **Seven years** — the longest in the system, to support any future dispute |
| **Audit log** | **Seven years** |

---

## I.6 Encryption summary

| What | How |
|---|---|
| Traffic | Modern transport encryption, all hops |
| Database at rest | Provider-managed disk encryption |
| Object storage at rest | Server-side encryption |
| Third-party credentials | Envelope encryption with per-workspace keys and context binding |
| Second-factor secrets | Envelope encryption |
| Session tokens | Hashed at rest, never stored in plaintext |
| Passwords | Memory-hard hashing, never encryption — they must not be reversible |
| Recovery codes | Hashed |
| Backups | Encrypted at rest and in transit |
| Addresses in logs | Hashed rather than stored raw |

---

## I.7 Rate limiting

| Scope | Purpose |
|---|---|
| Unauthenticated endpoints, per address | Credential-stuffing defence |
| Authenticated reads, per user | Abuse and runaway-client defence |
| Authenticated writes, per user | Abuse defence |
| **Spending operations, per workspace** | **Cost protection — the most important limit in the system** |
| Concurrent runs, per workspace | Resource protection |
| Outbound per provider, per workspace | Quota protection with a reserved allocation for critical operations |
| Per AI tier | Token-rate and concurrency budgets |

**Rate limiting here is a financial control, not just an availability control.** A runaway loop calling artwork generation could spend a great deal of money very quickly. Concurrency limits on the artwork queue and per-workspace spending limits bound that exposure structurally.

---

## I.8 Secure integration practices

| Practice | Detail |
|---|---|
| Authorisation flow | Proof-key exchange; state verified on callback; any mismatch aborts before token exchange |
| Token storage | Encrypted, never returned to the client |
| Token refresh | Proactive, with a distributed lock preventing concurrent refresh from invalidating credentials |
| Scope | Minimum necessary, displayed before authorisation |
| Webhook verification | Signature checked with constant-time comparison; timestamp window enforced; event identifiers deduplicated against a replay cache |
| Webhook handling | Verify, persist, enqueue, return. **No work inside the request.** |
| Response validation | Every provider response validated against a schema; unexpected shapes are errors, not silent misinterpretation |
| Outbound request safety | Any address derived from user or imported data is validated against an allowed list of schemes and hosts; private and internal ranges are blocked at resolution; cross-origin redirects are not followed |
| Egress control | Worker processes may reach only an explicit allowed list of provider hosts; anything else is denied and logged as an anomaly |

**The outbound address validation deserves emphasis.** Imported files can contain image addresses. Fetching an operator-supplied address without validation would allow a crafted file to make the server request internal resources. Validation against an allowed list, with private ranges blocked at resolution and redirects constrained, closes this.

---

## I.9 File security

| Control | Detail |
|---|---|
| Upload path | Direct browser to storage via a signed target — **bytes never pass through the application** |
| Signed target scope | Restricted to one object key, one content type and a maximum size, expiring in minutes |
| Object keys | Generated server-side from identifiers. **User input never contributes to a storage path.** |
| Type verification | Determined by inspecting the file's actual leading bytes, never by trusting the declared type or the extension |
| Size limits | Enforced in the signed target and again on registration |
| Storage | Outside any executable path; there is no path by which a stored file could be executed |
| Serving | Signed links only, short-lived, with headers preventing browser interpretation of content |
| Access control | Every request verifies workspace ownership |
| Import safety | Streamed parsing with row limits — no full-file buffering, so a large file cannot exhaust memory |
| Export safety | Leading characters that spreadsheet software would interpret as formulas are neutralised |
| Archives | Not accepted, eliminating decompression-bomb risk entirely |

---

## I.10 Application security controls

| Vector | Control |
|---|---|
| Injection into queries | Parameterised access only; a lint rule bans unsafe raw query construction; the few places raw SQL is genuinely needed use parameter binding and are individually reviewed |
| Cross-site scripting | Framework escaping by default; raw HTML insertion banned by lint; model-generated text rendered as plain text or through a sanitising renderer with an allowed-element list |
| Content policy | Strict policy with per-request nonces; no inline or evaluated script permitted in production |
| Cross-site request forgery | Token verification on all cookie-authenticated mutations; the progress stream is read-only |
| Clickjacking | Framing denied |
| Content sniffing | Disabled by header |
| Referrer leakage | Constrained by policy |
| Deserialisation | Structured data only, schema-validated; no dynamic evaluation; no prototype-polluting merges |
| Dependency risk | Lockfile enforcement, automated scanning, generated software bill of materials, minimal dependency surface in adapters |

### AI-specific controls

Detailed in [Part F](#part-f--ai-architecture); the security-relevant summary:

1. Untrusted external text is quarantined in delimited blocks, appended after instructions, never interpolated into them.
2. Prompts consuming untrusted data have **no tool access** and produce constrained classification output only.
3. **The generative stage never receives raw external text** — only aggregated statistics. Injection therefore has no path to influence what the system creates.
4. Model output is never executed, and never constructs an address, query or path without validation against an allowed list.
5. Output values outside the allowed set are rejected, never coerced.
6. Spend is bounded before the call, so a prompt inducing long generation cannot produce unbounded cost.

---

## I.11 The legal gate as a security control

The legal and safety engine is treated as a security control, not a product feature:

- **Enforced in the service layer, where the external call is made** — not in the interface. A client bypassing the interface entirely still cannot generate artwork for a blocked concept.
- Overrides require step-up authentication, an exact typed confirmation, and a written justification — all permanently recorded.
- Blocked concepts have **no override path**. The endpoint rejects them unconditionally.
- A registry failure must never appear as a clean result; reduced coverage is recorded and reflected in the assessment.
- Screening records are retained for seven years.

---

## I.12 Audit logging

**Append-only.** Update and delete privileges are revoked at the database level, not merely avoided in code.

**Always recorded:** authentication events including failures and lockouts, session creation and revocation, credential connection, rotation and removal, budget changes, **every publish**, **every legal override**, scoring configuration activation and reversion, data export, and deletion requests.

**Each entry contains:** actor, action, entity type and identifier, before and after state, justification where applicable, hashed address, correlation identifier and timestamp.

**Integrity verification:** a nightly task verifies a rolling hash chain across each day's entries and alerts on any discontinuity.

---

## I.13 Monitoring and response

### Security-relevant alerts

| Signal | Severity |
|---|---|
| Repeated authentication failures against one account | High |
| Authentication from an unusual location | Medium |
| Credential decryption failure | High |
| Unexpected spending pattern | High |
| Publishing failure | Critical |
| Legal engine failure | Critical |
| Egress attempt to a non-allowed host | High |
| Audit chain discontinuity | Critical |
| Cross-workspace access attempt | Critical |

### Kill switches

Single-toggle controls, **tested quarterly**:

- Disable all publishing
- Disable all AI and image spending
- Disable all amber-tier data adapters
- Revoke all sessions
- Enter maintenance mode

### Incident severity

| Level | Definition | Response |
|---|---|---|
| **1** | Credential compromise, data breach, unauthorised marketplace write | Immediate: revoke all tokens, invalidate all sessions, disable publishing, preserve logs, notify within the hour, assess regulatory notification obligations |
| **2** | Contained unauthorised access; significant unexplained spend | Rotate affected credentials, audit blast radius, patch, report within a day |
| **3** | Vulnerability found, not exploited | Patch within the severity's window |
| **4** | Hygiene issue | Backlog with an owner and a date |

**Standing runbooks:** credential compromise, unexpected spend, unauthorised publish, provider breach notification, data restore, and isolation misconfiguration. Each names the detection signal, the containment step, the eradication step, the recovery step and the evidence to preserve.

---

## I.14 Verification

| Activity | Frequency | Gate |
|---|---|---|
| Dependency scanning | Every change and daily | Critical findings block |
| Static analysis with custom rules | Every change | High findings block |
| Secret scanning | Every change, plus full history periodically | Any hit blocks |
| Container scanning | Every build | Critical findings block deployment |
| Software bill of materials | Every release | Stored with the artefact |
| **Cross-workspace isolation suite** | **Every change** | Attempts access with another workspace's identifiers on every endpoint; any success blocks |
| Manual security review | Every phase gate | Sign-off required |
| Penetration test | Before multi-user launch, annually thereafter | Findings tracked to closure |
| Restore drill | Monthly | Result recorded |
| Kill-switch test | Quarterly | Result recorded |

**Custom static-analysis rules enforce the architecture's security invariants:** no vendor library outside its adapter, no raw HTML insertion, no unsafe query construction, no credential access outside the credential service, and no outbound request to a host constructed from a variable without validation.

---

# Part J — Scalability Architecture

## J.1 Growth stages

| Stage | Workspaces | Runs/day | Listings tracked | Shape |
|---|---|---|---|---|
| **1 — Personal** | 1 | 3 | 1,000 | One web instance, one worker, one database, one cache |
| **2 — Early multi-user** | ~50 | 150 | 50,000 | A few web instances, several workers by class, database plus replica |
| **3 — Growth** | ~1,000 | 3,000 | 1,000,000 | Autoscaled web, queue-sharded workers, connection pooling, partitioned hot tables, storage tiering |
| **4 — Scale** | ~10,000 | 30,000 | 10,000,000 | Regional deployment, database sharding by workspace, dedicated analytical store, external workflow engine |

**Each stage is reached by adding capacity or splitting an existing seam — never by rewriting.**

---

## J.2 What actually breaks first

Ordered by when it bites, not by how interesting it is. **This ordering is the most useful thing in this section**, because most scaling plans get it backwards.

| # | Bottleneck | Bites at | Signal | Response |
|---|---|---|---|---|
| **1** | **Third-party rate limits** | Stage 2 | Rising rate-limit responses, growing queue age | Per-workspace credentials — quota then scales with tenant count rather than being a shared ceiling |
| **2** | **AI cost per run** | Stage 2 | Gross margin per workspace falling | Tier discipline, aggregation, prompt caching, and **shared market-fact caching** |
| 3 | Worker CPU during analysis and image processing | Stage 2–3 | Queue age, CPU saturation | Horizontal worker scaling; separate worker class for image processing |
| 4 | Database write throughput on observation tables | Stage 3 | Write latency, maintenance lag | Partitioning (already in place), batched writes, bulk loading, archival tiering |
| 5 | Database connection count | Stage 3 | Connection saturation | Transaction-mode connection pooling |
| 6 | Analytical query latency | Stage 3 | Report page latency | Read replicas, materialised aggregates, pre-computed report artefacts |
| 7 | Vector index memory | Stage 3–4 | Memory pressure, recall degradation | Partition vectors by workspace; quantisation; a dedicated service only if memory-bound |
| 8 | Object storage volume | Stage 3–4 | Cost | Lifecycle tiering, deduplication, thumbnail-only retention for competitor images |
| 9 | Orchestration coordination | Stage 4 | Scheduling latency, lock contention | Migrate the step machine to an external workflow engine — the graph definition ports directly |

**The first two are external and economic, not internal and technical.** For this product, the limiting factor is provider quota and AI cost long before it is CPU or database throughput. That is why the cost engineering in Part F and the per-workspace credential model below matter more than any sharding plan.

---

## J.3 Multi-user support

### What is already done

| Requirement | Status |
|---|---|
| Workspace identifier on every table | **Done from the first migration** |
| Composite indexes leading with it | Done |
| Membership table with role enum | Done — one row |
| Isolation policies written and tested | Done — enforcement switched on later |
| Per-workspace credential encryption | Done |
| Per-workspace rate-limiting keys | Done |
| Per-workspace budgets and cost ledgers | Done |
| Audit log with actor attribution | Done |
| Stateless web and worker tiers | Done |
| Cursor pagination everywhere | Done |

### What multi-user adds

| Step | Work |
|---|---|
| Enable isolation enforcement | Switch the database role. The policies already exist and are already tested. |
| Invitations and role enforcement | Invitation flow, membership management, enforce the existing permission matrix |
| Per-workspace provider credentials | Structurally supported already; this is a user-experience change |
| Fair scheduling | Per-workspace sub-queues with weighted round-robin and plan-derived concurrency caps |
| Workspace switching | Session-scoped active workspace |

**The founding operator becomes the first tenant with zero migration** — their workspace row already exists, their data is already scoped, their credentials are already per-workspace encrypted.

---

## J.4 User isolation

Isolation is enforced at three independent layers, so a failure at any one does not expose data.

| Layer | Mechanism |
|---|---|
| **Application** | Workspace resolved server-side from the session; every repository method scoped; every entity identifier re-resolved |
| **Database** | Row-level isolation policies keyed to a session variable set per connection checkout |
| **Storage** | Object keys namespaced by workspace; signed links scoped to a single object; ownership verified before signing |

**Verification is continuous, not periodic.** An automated suite attempts, for every endpoint, to access an entity belonging to a different workspace, and asserts a *not found* response with normalised timing. This runs on every change, not just at audit time.

**Additional isolation controls:** per-workspace queue partitioning and concurrency caps so one tenant cannot starve others; per-workspace rate limits and spending budgets so one tenant cannot exhaust shared provider quota or generate unbounded cost.

---

## J.5 Subscription system preparation

The billing system is **not built** in Phase 1, but the ground is prepared so that adding it is additive.

| Prepared now | How it is used later |
|---|---|
| Workspace has a plan field | Becomes the subscription tier |
| Per-workspace cost ledger at call granularity | Becomes usage metering with no new instrumentation |
| Budget guard middleware on every spending operation | Plan entitlement checks slot in beside the existing budget check |
| Per-workspace concurrency limits | Become plan-derived limits |
| Workspace status field | Supports suspension for non-payment |

### What billing adds

| Component | Purpose |
|---|---|
| Plans | Price points with entitlement definitions |
| Subscriptions | Per-workspace state, period, cancellation intent |
| Usage events | Metered quantities per billing period, with our marginal cost recorded alongside for margin analysis |
| Entitlement counters | Fast counters for enforcement without aggregating the event stream |
| Payment provider webhooks | Lifecycle synchronisation |

**Two design commitments made now that matter later:**

1. **Usage-based components are essential, not optional.** Every run and every artwork costs real money. A flat unlimited plan is how products with per-unit costs go bankrupt. Because cost is already tracked per call and attributed per workspace, metering requires no new instrumentation.
2. **Non-payment downgrades to read-only. It never deletes data.** The operator's research history and artwork are their asset; withholding access is legitimate, destroying it is not.

---

## J.6 Database scaling

### Partitioning — in place from day one

Five tables will dominate row count and are declared as time-partitioned from the first migration, even with a single partition present. A maintenance task pre-creates upcoming partitions and archives expired ones.

**Cost of doing this now: one migration. Cost of doing it at a billion rows: an outage.**

### Read replicas — Stage 2

| Workload | Target |
|---|---|
| Interactive report and dashboard reads | Replica |
| Analytics and exports | Replica |
| Run orchestration, all writes, read-after-write paths | Primary |

**Routing is explicit at the repository layer, never inferred.** Replica lag silently breaking a read-after-write invariant is a class of bug that is extremely difficult to diagnose, and explicit routing prevents it by construction.

### Connection pooling — Stage 3

Transaction-mode pooling with per-service pool ceilings. The data access layer is configured for the constraints this imposes — no session-level state, no reliance on prepared-statement caching across transactions.

### Sharding — Stage 4

**Workspace identifier is the natural shard key, and this is a direct consequence of the day-one tenancy decision.**

There are **no cross-workspace queries anywhere in the application**, with the single exception of platform-level administrative analytics, which move to a separate analytical store. That property is what makes sharding tractable rather than a rewrite.

Approach, in order of preference: distributed tables keyed on workspace, then logical sharding with a routing layer, then regional isolation. Shared reference data — product catalogue, providers, taxonomy — is replicated to every shard.

### Archival tiering — Stage 3

Raw observations beyond the retention window move to columnar files in object storage, queryable on demand. **Derived analysis stays in the primary database indefinitely** — it is small, and it is what the product actually reads.

---

## J.7 Storage scaling

| Concern | Approach |
|---|---|
| Deduplication | Content-addressed keys mean identical images are stored once regardless of how many listings or workspaces reference them |
| Lifecycle tiering | Artwork originals move to infrequent-access tiers after a period; competitor images expire on schedule; mockups archive after delisting |
| Retention discipline | Competitor images are thumbnails only — full-resolution copies are never retained |
| Cost profile | Egress-free storage means repeated viewing does not scale cost with usage |
| Regional placement | At Stage 4, buckets follow the workspace's region |

---

## J.8 AI usage scaling

**This is where unit economics are won or lost.**

### The per-workspace credential option

At multi-user scale, higher tiers may supply their own AI and image generation keys. This converts the largest variable cost into a customer-borne cost for the heaviest users, and is straightforward because credentials are already per-workspace.

### Shared market-fact caching — the single largest economic lever

Two workspaces researching the same niche in the same week should not both pay to analyse the same four hundred competitor images.

```
   ┌──────────────────────────────────────────────────────┐
   │  SHARED — market facts, content-addressed             │
   │  • image content hash → visual attributes            │
   │  • normalised term → registry lookup result          │
   │  Not owned by any workspace. No workspace-derived    │
   │  information. Pure, verifiable market facts.         │
   └──────────────────────────────────────────────────────┘
                            │
   ┌────────────────────────┴──────────────────────────────┐
   │  TENANT-SCOPED — everything else                      │
   │  cohorts · findings · gaps · concepts · scores ·      │
   │  artwork · listings · outcomes · configurations       │
   └───────────────────────────────────────────────────────┘
```

**The boundary is enforced by table separation, not by a query filter.** Shared tables contain only content-addressed market facts. Anything derived from a workspace's own configuration, cohort definitions or history remains tenant-scoped and is never shared.

**Effect:** at Stage 3 this roughly halves the dominant marginal cost. It is achievable because visual analysis is already keyed by image content hash rather than by listing identity — a schema decision made in Phase 1 specifically to enable this.

### Other AI scaling controls

| Control | Effect |
|---|---|
| Aggregation before prompting | The largest single reduction, applied everywhere |
| Tier discipline | The cheapest capable tier for each task, justified by evaluation |
| Prompt caching | Shared context reused across sibling calls in a run |
| Concept-before-artwork gating | Only human-approved concepts consume image budget |
| Per-workspace spend caps | Bound worst-case exposure per tenant |
| Batch sizing | Tuned per model as pricing and limits change |

---

## J.9 Provider quota scaling

**The key structural insight:** in multi-user mode, each workspace connects **their own** marketplace and fulfilment credentials. Provider quota therefore scales with tenant count rather than being a shared ceiling.

| Provider | Single user | Multi-user |
|---|---|---|
| Marketplace | Operator's own application quota | Per-workspace, with reserved publishing allocation per workspace |
| Fulfilment | Operator's token | Per-workspace token |
| Image generation | Shared platform key | Shared with per-workspace metering, or workspace-supplied on higher tiers |
| AI | Shared platform key | Shared with metering, or workspace-supplied on higher tiers |
| Market data | Operator's own subscription and exports | Per-workspace; the file-import path scales trivially |

**Per-workspace rate limiting is keyed this way from day one**, so this transition requires no code change — only the credential source changes.

---

## J.10 Queue and worker scaling

| Stage | Shape |
|---|---|
| 1 | One worker process handling all queues |
| 2 | Separate worker classes: analysis, vision, artwork, publishing, synchronisation |
| 3 | Per-class autoscaling driven by queue age; per-workspace fair scheduling |
| 4 | Regional queue clusters with workspace-affinity routing for cache locality |

### Fair scheduling — built at Stage 2, not Stage 3

One workspace running fifty deep research runs must not starve everyone else. Implemented as weighted round-robin across per-workspace sub-queues with plan-derived concurrency caps.

**This is built early deliberately**, because it is the multi-tenant behaviour most likely to be discovered too late — typically during the first incident where one customer's usage degrades everyone else's experience.

### Scaling signals

| Queue | Signal | Correct response |
|---|---|---|
| Analysis, vision | Queue age rising | Add workers |
| **Artwork** | Queue age rising | **Investigate first** — usually a retry loop or a cost problem, not a capacity problem |
| **Publishing** | Queue age rising | **Do not add workers** — almost always upstream rate limiting, and more workers make it worse |

---

## J.11 Web tier scaling

Fully stateless. Sessions live in the shared store; no mutable state is held in process. Scaling is horizontal on request latency or CPU.

The progress stream fans out through shared publish-subscribe, so **any instance can serve any run's stream** — no session affinity is required, which is what allows instances to be added and removed freely.

---

## J.12 Cost trajectory

| Stage | Workspaces | Infrastructure | AI and image | Gross margin |
|---|---|---|---|---|
| 1 | 1 | Low fixed | Low variable | n/a |
| 2 | ~50 | Modest | Significant | Thin — investment phase |
| 3 | ~1,000 | Moderate | **Dominant** | Acceptable |
| 3 with shared market caching | ~1,000 | Moderate | Substantially reduced | **Roughly doubles** |
| 4 | ~10,000 | Significant | Dominant but efficient | Healthy |

**The single most important number here is the effect of shared market-fact caching at Stage 3.** It approximately doubles gross margin. That is why the caching boundary is an architectural priority rather than an optimisation, and why usage-based pricing components are necessary to prevent heavy users from destroying unit economics.

---

## J.13 What we deliberately do not build early

| Deferred | Reasoning |
|---|---|
| Microservices | Module boundaries already exist where service boundaries would go. Distributed transactions and operational overhead would slow delivery for no current benefit. Split at Stage 4, along the existing seams. |
| Container orchestration platform | Managed platforms carry Stages 1–3 comfortably. Adopt when multi-region or bespoke scheduling forces it. |
| Event sourcing | Immutable snapshots and the append-only step model already provide auditability and replay where it matters, without read-model complexity. |
| Dedicated vector database | The database extension handles millions of vectors. Move only when index memory, not query latency, is the binding constraint. |
| Data warehouse | Replicas answer analytical queries through Stage 3. Introduce a columnar store when analytical load measurably affects transactional latency. |
| External workflow engine | The custom orchestrator is small and fits the domain exactly. Adopt at Stage 4 when cross-region durability justifies the operational cost — the graph definition ports directly. |

**Each deferral has a stated trigger.** These are not permanent refusals; they are decisions with documented conditions for revisiting.

---

## J.14 Capacity planning

| Signal | Threshold | Action |
|---|---|---|
| Queue age, ninety-fifth percentile | Beyond a few minutes | Add workers to that class |
| Database CPU | Sustained high | Add a replica, or optimise the highest-total-time query |
| Connection utilisation | Approaching the ceiling | Introduce connection pooling |
| Storage growth rate | Above trend | Review retention policy |
| Cost per run | Drifting above budget | Prompt and tier review |
| Vector index memory | Approaching available memory | Partition by workspace; consider quantisation |
| Interactive latency, ninety-fifth percentile | Beyond budget | Profile; add web capacity if it is throughput-related |

### Continuous load verification

| Test | Frequency | Assertion |
|---|---|---|
| Sustained load | Nightly | Latency within budget at several times current peak |
| Spike | Weekly | Large short burst degrades gracefully, queue absorbs, no errors |
| Soak | Weekly | No memory growth, no connection leak, no queue drift over hours |
| Data volume | Monthly | Reports over a much larger seeded dataset meet latency budgets |
| **Chaos** | **Weekly** | **Worker killed mid-run produces identical output with zero duplicate external effects** |
| Provider degradation | Weekly | Injected failures open breakers, pause runs, lose no data |

---

# Part K — Project Structure

## K.1 Repository shape

**A single repository with workspace packages and a build orchestrator.**

**Why one repository.** The scoring, statistical, pricing and validation logic is used by both the browser and the worker. Splitting repositories would force premature package publishing and version skew between a user interface and a pipeline that must stay in lockstep. There is one team and one release cadence.

```
pod-intelligence/
├── apps/                    deployable processes
│   ├── web/                 user interface, API surface, webhook receivers
│   ├── worker/              queue consumers, pipeline execution
│   └── scheduler/           cron leader
│
├── packages/                shared libraries
│   ├── contracts/           shared types, enums, validation schemas
│   ├── domain/              pure logic — scoring, statistics, pricing, rules
│   ├── database/            schema, repositories, migrations
│   ├── queue/               queue abstraction, job registry
│   ├── orchestrator/        step graph, runner, budget guard, recovery
│   ├── engines/             one package per domain service
│   ├── adapters/            one package per external provider
│   ├── ai/                  model gateway, prompt registry, evaluations
│   ├── observability/       logging, tracing, metrics, cost telemetry
│   ├── ui/                  shared components and design tokens
│   └── config/              environment schema, feature flags, model bindings
│
├── docs/                    this specification set
├── infra/                   infrastructure definitions, containers, scripts
├── tooling/                 shared lint, type and build configuration
├── tests/                   end-to-end and load suites
└── (workspace and build configuration at the root)
```

---

## K.2 Frontend structure

```
apps/web/
├── src/
│   ├── app/                              routing tree
│   │   ├── (auth)/                       login · second factor · recovery
│   │   └── (app)/                        authenticated application
│   │       ├── (dashboard)                dashboard
│   │       ├── research/
│   │       │   ├── new/                   run wizard
│   │       │   └── [runId]/
│   │       │       ├── (monitor)          live progress
│   │       │       ├── opportunity/
│   │       │       ├── competitors/
│   │       │       ├── success/
│   │       │       ├── failure/
│   │       │       ├── gaps/
│   │       │       ├── concepts/
│   │       │       └── compare/[otherId]/
│   │       ├── concepts/[conceptId]/      detail, prediction, legal
│   │       ├── artwork/[artworkId]/       studio
│   │       ├── products/[draftId]/
│   │       │   ├── (tabs)                 product · pricing · seo · fulfilment · marketplace
│   │       │   └── review/                final approval
│   │       ├── listings/[listingId]/
│   │       ├── analytics/                 performance · accuracy · costs · learning
│   │       ├── niches/
│   │       └── settings/                  account · workspace · integrations ·
│   │                                      economics · scoring · data · notifications
│   │
│   ├── api/                              server endpoints
│   │   ├── (rpc handler)
│   │   ├── (progress stream)
│   │   ├── webhooks/                     fulfilment · marketplace · billing
│   │   ├── auth/                         authorisation callbacks
│   │   └── health/
│   │
│   ├── server/                           server-only code
│   │   ├── api/
│   │   │   ├── router.ts                 composition
│   │   │   ├── procedures.ts             the endpoint classes from Part D
│   │   │   ├── context.ts
│   │   │   └── routers/                  one file per resource group
│   │   ├── services/                     use-case coordination — thin
│   │   ├── auth/                         session, second factor, password
│   │   └── cache-map.ts                  mutation → invalidated queries
│   │
│   ├── components/
│   │   ├── scores/                       dial, breakdown, contribution waterfall
│   │   ├── evidence/                     finding card, evidence drawer, confidence chip
│   │   ├── runs/                         progress tracker, cost meter, step list
│   │   ├── tables/                       virtualised data table, column config, export
│   │   ├── charts/                       distribution, scatter, radar, bubble, series
│   │   ├── concepts/                     board, card, detail drawer
│   │   ├── artwork/                      canvas, variant strip, quality panel
│   │   ├── products/                     configuration table, pricing panel, checklist
│   │   ├── seo/                          variation editor, tag input, preview
│   │   ├── layout/                       shell, navigation, top bar
│   │   └── forms/                        validated inputs, money input, wizard
│   │
│   ├── hooks/                            data access, progress stream, selection
│   ├── stores/                           wizard, selection, drawer — nothing else
│   ├── lib/                              formatting, URL state, utilities
│   ├── styles/                           tokens, theme
│   └── messages/                         all user-facing strings
└── tests/
```

### Frontend conventions

| Rule | Reason |
|---|---|
| Server components by default; interactivity marked explicitly and pushed to leaves | Minimises shipped JavaScript on data-dense pages |
| Every route has loading and error boundaries; report sections have independent boundaries | A slow section never blocks the page |
| **No business logic in components** | Components render. Server services coordinate. The domain package computes. |
| All user-facing strings in the message catalogue from day one | Localisation readiness costs nothing now |
| Filters, sort, tab and pagination in the URL | Every view is shareable and survives refresh |
| One cache map declaring every mutation's invalidations | Cache correctness reviewable in one file |

---

## K.3 Backend structure

```
apps/worker/
├── src/
│   ├── index.ts                   boot, telemetry, queue registration, graceful shutdown
│   ├── health.ts                  container health endpoint
│   ├── processors/                one per job type
│   │   ├── run.start · run.step
│   │   ├── vision.analyse · embedding.compute
│   │   ├── artwork.generate · artwork.process
│   │   ├── publish.fulfilment · publish.marketplace
│   │   ├── sync.performance · sync.catalogue
│   │   ├── learning.evaluate
│   │   └── maintenance.*          reaper · retention · consistency · partitions
│   ├── lifecycle/                 signal handling, drain, worker leases
│   └── registry.ts                queue-to-processor binding

apps/scheduler/
└── src/
    ├── index.ts                   leader lock, tick loop
    └── jobs.ts                    schedule definitions — all idempotent
```

---

## K.4 Database structure

```
packages/database/
├── schema/                        schema definition, source of truth
├── migrations/                    forward-only, timestamped, reviewed
│   └── (each with an up script and a documented rollback or
│         an explicit forward-only note with its compensating action)
├── seed/                          realistic development fixture
└── src/
    ├── client.ts                  read and write clients, isolation session variables
    ├── repositories/              ONE PER AGGREGATE — the only place the ORM is called
    │   ├── run · niche · shop · listing · analysis
    │   ├── concept · artwork · product · seo
    │   ├── listing · performance · outcome
    │   └── file · integration · audit
    ├── transactions.ts
    ├── partitions.ts              raw SQL — creation and rotation
    └── vector.ts                  raw SQL — similarity queries
```

### The seed fixture is a deliberate investment

The development seed produces a **complete, realistic workspace**: a finished run with ten shops, several hundred listings with full visual analysis, all four reports, twenty concepts, several artworks, two product drafts and two published listings with performance history.

**Why this matters more than it appears.** Without it, every developer builds every screen against an empty state, discovers layout and performance problems only in production, and cannot work on later-stage features without manually producing earlier-stage data. With it, every screen has realistic content from the first minute.

---

## K.5 AI modules

```
packages/ai/
├── src/
│   ├── client.ts                  the gateway — the only path to any model
│   ├── tiers.ts                   capability tier → model binding, from configuration
│   ├── contracts.ts               output contract enforcement, repair, retry
│   ├── quarantine.ts              untrusted content wrapping and sanitisation
│   ├── cache.ts                   input-hash caching, per policy
│   ├── budget.ts                  reservation and reconciliation
│   ├── pricing.ts                 cost table
│   └── telemetry.ts               usage recording
└── evaluations/
    ├── runner.ts
    ├── suites/                    contract · groundedness · constraints ·
    │                              injection · safety · distinctness · quality
    └── fixtures/                  golden sets, adversarial cases

packages/engines/<engine>/prompts/<prompt-id>/
├── v<version>/
│   ├── template                   the prompt
│   ├── contract                   the output schema
│   ├── metadata                   purpose, tier, temperature, ceiling, cache policy
│   └── changelog
└── evaluations/
    ├── cases
    └── rubric
```

### Prompt organisation rules

| Rule | Reason |
|---|---|
| Prompts are files, never inline strings | Reviewable, versionable, testable |
| Semantic versioning | Output-shape changes are major; behavioural changes are minor |
| Every generated entity records its prompt version | A prompt change never retroactively alters stored outputs |
| Registry validated at process start | A missing contract fails startup, not production |
| Every prompt has an evaluation suite | An AI feature without an evaluation is not shippable |

---

## K.6 Integration modules

```
packages/adapters/<provider>/
├── src/
│   ├── index.ts                   the public interface — the only export
│   ├── client.ts                  transport with auth, retry, limits, breaker, tracing
│   ├── schemas.ts                 response validation for every endpoint
│   ├── mappers.ts                 provider shapes ↔ domain types
│   ├── errors.ts                  provider errors → internal catalogue with remedies
│   └── capabilities.ts            what this adapter can do
├── fixtures/                      recorded responses, including every documented error
└── tests/
```

**Adapters present:** ai · image-generation · marketplace · fulfilment · market-data · trademark · storage · email.

**The market-data adapter is the exception in shape**, containing the provider chain described in Part G:

```
packages/adapters/market-data/src/
├── index.ts                       the chain resolver
├── merge.ts                       field-level merge by confidence
├── provenance.ts                  per-field source tracking
└── providers/
    ├── operator-file/             DEFAULT — always available
    ├── provider-api/              if one exists
    ├── assisted-session/          OPT-IN, off by default, no evasion capability
    ├── public-marketplace/        always available
    └── fixture/                   development and testing
```

---

## K.7 The domain package — the crown jewels

```
packages/domain/src/
├── scoring/
│   ├── opportunity.ts             five sub-scores, combination, verdict banding
│   ├── shop-selection.ts          selection score including the age preference
│   ├── design-success.ts          five prediction dimensions, contribution vector
│   ├── gap.ts                     gap score and the demand floor
│   ├── product.ts                 configuration scoring
│   ├── seo-quality.ts             variation quality
│   ├── normalise.ts               scaling functions
│   └── bands.ts                   score → band mapping
├── statistics/
│   ├── cohorts.ts                 age-normalised partitioning
│   ├── contingency.ts             prevalence, lift, significance
│   ├── effect-size.ts             rank-based effect measures
│   ├── correlation.ts             rank correlation, multicollinearity
│   ├── bands.ts                   optimal numeric band search
│   ├── interactions.ts            pairwise interaction detection
│   └── confidence.ts              sample size and significance → confidence level
├── pricing/
│   ├── fee-stack.ts               full fee model, both advertising cases
│   ├── solve.ts                   target margin ↔ fixed price
│   ├── margin.ts                  floor evaluation
│   └── money.ts                   minor-unit arithmetic, rounding, currency
├── text/                          normalisation, phrases, weighting, similarity, validators
├── colour/                        perceptual space, clustering, family mapping, gamut
├── placement/                     per-product-type artwork positioning
├── quality/                       print-readiness criteria evaluation
├── legal/
│   └── risk-rules.ts              the deterministic risk decision table
└── learning/                      feature assembly, shrinkage, back-test measures
```

### The rules that make this package special

| Rule | Enforcement |
|---|---|
| **Imports nothing but shared types** | Build-time dependency analysis |
| **No input or output of any kind** | No database, no network, no file access |
| **No ambient time or randomness** | Both injected, so tests are deterministic |
| **Every function is pure** | Same inputs always produce the same outputs |
| **Highest test coverage in the repository** | Including property-based tests for monotonicity, bounds and invariance |

**Why this discipline is worth the constraint.** This package contains the product's actual intellectual property. It is what makes scores reproducible, the learning loop auditable, and the entire analytical core testable in milliseconds without a network. Every rule above exists to protect one of those three properties.

---

## K.8 Testing structure

```
packages/domain/**/tests/           unit and property tests
packages/adapters/*/tests/          fixture-backed integration
packages/engines/*/tests/           engine tests with mocked adapters
packages/ai/evaluations/            AI evaluation suites
apps/web/tests/                     component and route tests
tests/e2e/                          end-to-end journeys
tests/load/                         load scenarios
tests/chaos/                        failure injection
```

| Layer | Target | Frequency |
|---|---|---|
| Pure domain | Highest coverage; **every scoring function tested exhaustively**, including property tests | Every change |
| Adapters | Every endpoint and every documented error, against recorded fixtures | Every change |
| Engines | Happy path plus **each degradation path** | Every change |
| AI evaluations | Fast suite per change; full suite nightly | Both |
| End-to-end | Five critical journeys: run research · select concepts · generate artwork · create a draft · publish | Before merge |
| **Chaos** | **Worker killed mid-run produces identical output with zero duplicate effects** | Weekly |
| Load | Latency budgets under multiples of peak | Nightly |
| **Isolation** | **Cross-workspace access attempts on every endpoint** | Every change |

**No test in the build pipeline ever calls a live external service.** Every adapter has recorded fixtures. This makes the suite fast, deterministic, free and runnable offline.

---

## K.9 Documentation structure

```
docs/
├── PART-1-product-definition.md            what and why
├── PHASE-1B-technical-architecture.md      this document
├── adr/                                    architecture decision records
├── runbooks/                               one per alert — an alert without one is a defect
├── prompts/                                prompt design guidance and evaluation results
├── api/                                    generated interface reference
└── operations/                             environment setup, release process, on-call
```

---

## K.10 Enforced boundaries

Dependency rules are checked in the build, not left to convention. Convention erodes within a quarter.

```
   apps/*             → packages/*                              allowed
   engines            → domain, database, adapters, ai,
                        contracts, queue, observability          allowed
   domain             → contracts                                allowed — ONLY
   domain             → database, adapters, network, filesystem  FORBIDDEN
   adapters           → contracts, observability                 allowed
   adapters           → engines, database                        FORBIDDEN
   engines            → engines                                  FORBIDDEN
   anything           → another package's internal paths         FORBIDDEN
   vendor library     → outside its own adapter                  FORBIDDEN
   ORM                → outside repositories                     FORBIDDEN
```

**A violation fails the build.** This rule file is the architecture's enforcement mechanism — without it, the layering described throughout this document degrades into folders that merely look organised.

---

## K.11 Naming conventions

| Item | Convention | Example |
|---|---|---|
| Files | Hyphenated lowercase | `success-attributes` |
| Components | Capitalised, file matches export | `ScoreDial` |
| Hooks | Prefixed with use | `useRunProgress` |
| Types | Capitalised, no prefix letter | `SuccessAttribute` |
| Validation schemas | Type name plus suffix | `SuccessAttributeSchema` |
| Enum values | Lowercase with underscores, matching the database | `muted_green` |
| Database tables | Lowercase with underscores, plural | `success_attributes` |
| Database columns | Lowercase with underscores | `cohort_support_percent` |
| Queue jobs | Dot-namespaced | `artwork.generate` |
| API procedures | Verb-first, camel case | `regenerateOne` |
| Prompts | Hyphenated with a version | `concept-generation-success/v1.2.0` |
| Feature flags | Domain-prefixed with underscores | `engine_gap_v2` |
| Environment variables | Uppercase with underscores | `RUN_CONCURRENCY_LIMIT` |
| **Money fields** | **Amount plus currency pair** | `retail_price_amount`, `retail_price_currency` |
| Booleans | Prefixed with is, has or can | `is_bestseller` |
| Timestamps | Suffixed with at | `published_at` |

---

## K.12 Where a new feature goes

A worked example — adding a bundle-recommendation capability:

| Step | Location |
|---|---|
| 1. Shared types | `packages/contracts` |
| 2. Scoring arithmetic and tests | `packages/domain/scoring` |
| 3. Engine with its prompts and evaluations | `packages/engines/bundle` |
| 4. Schema change and repository | `packages/database` |
| 5. Step definition | `packages/orchestrator` |
| 6. Job processor registration | `apps/worker` |
| 7. API endpoints | `apps/web/src/server/api/routers` |
| 8. Screens and components | `apps/web/src/app` |
| 9. Feature flag, default off | Configuration |
| 10. Documentation and a decision record if a boundary moved | `docs/` |

**Ten mechanical steps, each in an obvious place.** That predictability is the purpose of the structure.

---

# Part L — Deployment Architecture

## L.1 Environments

| Environment | Purpose | Data | Providers | Access |
|---|---|---|---|---|
| **Local** | Development | Seeded fixture | **All mocked** | Developer machine |
| **Test** | Build pipeline | Ephemeral per run | All fixtures | Pipeline only |
| **Preview** | Per-change review | Branched fixture data | Mocked, except a sandbox shop | Team, protected |
| **Staging** | Pre-production verification | Synthetic, production-shaped | **Real**, with sandbox accounts and capped budgets | Team |
| **Production** | Live | Real | Real | Operator |

**Production data is never copied downward.** Staging is seeded by a generator that reproduces production's shape and volume without its content. This removes an entire category of data-handling risk from the development process.

---

## L.2 Development environment

### One command to a working system

```
   start the local stack     →  database, cache, object storage,
                                mail catcher, trace collector
   apply schema and seed     →  a complete realistic workspace
   run all processes         →  web, worker and scheduler with hot reload
```

### Fully offline

With provider mocking enabled, **every adapter is replaced by a fixture-backed implementation with configurable latency and failure injection.** The entire pipeline — research, analysis, concept generation, artwork, publishing — runs end to end in seconds, offline, at zero cost.

**This is one of the highest-leverage decisions in the whole architecture.** It means a developer can work on the publishing flow without a marketplace account, iterate on report layouts without paying for research, and reproduce a failure deterministically instead of hoping it recurs.

A failure-injection script exercises the resilience paths on demand: provider outages, slow responses, rate limiting, worker termination mid-step.

### Local configuration

Environment variables validated at start-up against a schema. The process refuses to start on invalid configuration rather than failing later. **Mocking cannot be enabled when the environment is production** — that combination is rejected outright.

---

## L.3 Testing environment

The build pipeline provisions an ephemeral database and cache per run, applies migrations, and executes:

```
   lint and type check
        ↓
   dependency boundary analysis          ← the architecture enforcement
        ↓
   unit and property tests               ← coverage gate on the domain package
        ↓
   adapter integration tests             ← fixtures only, never live services
        ↓
   AI evaluation fast suite              ← contract, groundedness, injection, cost
        ↓
   security scanning                     ← static analysis, dependencies, secrets, containers
        ↓
   build web and worker
        ↓
   end-to-end journeys                   ← mocked providers
        ↓
   isolation suite                       ← cross-workspace access attempts
        ↓
   performance budget check
```

**Gates that block a merge:** any lint or type error, coverage below the domain threshold, any failing test, a critical security finding, any secret detected, an AI evaluation regression, a performance budget breach, or a dependency boundary violation.

---

## L.4 Production environment

```
                        ┌──────────────────────┐
                        │    EDGE / CDN        │
                        │  TLS · protection ·  │
                        │  static caching      │
                        └──────────┬───────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │        WEB TIER             │
                    │  managed platform,          │
                    │  atomic deploys,            │
                    │  instant rollback,          │
                    │  autoscaled                 │
                    └──┬────────┬────────┬────────┘
                       │        │        │
        ┌──────────────▼──┐  ┌──▼─────┐  └────────┐
        │   DATABASE      │  │ CACHE  │           │
        │ managed,        │  │ queue, │           │
        │ point-in-time   │  │ limits │           │
        │ recovery,       │  └──▲─────┘           │
        │ replica at      │     │                 │
        │ stage 2         │     │                 │
        └──────────────▲──┘     │                 │
                       │        │                 │
                    ┌──┴────────┴──┐    ┌─────────▼────────┐
                    │  WORKER TIER │    │  OBJECT STORAGE  │
                    │  containers, │    │  private,        │
                    │  rolling     │    │  versioned,      │
                    │  restart     │    │  lifecycle rules │
                    └──▲───────────┘    └──────────────────┘
                       │
                    ┌──┴───────────┐
                    │  SCHEDULER   │
                    │  singleton,  │
                    │  lock-based  │
                    │  failover    │
                    └──────────────┘

        ┌──────────────────────────────────────────────┐
        │  OBSERVABILITY: logs · traces · metrics ·    │
        │  errors · cost telemetry — from all tiers    │
        └──────────────────────────────────────────────┘
```

### Recommended hosting

| Tier | Recommendation | Reasoning |
|---|---|---|
| Web | Vercel | Purpose-built for the framework; atomic deploys; edge caching; preview per change |
| Worker and scheduler | Fly.io or Railway | Long-running processes with memory for image work. Serverless is the wrong shape for a fifteen-minute step. |
| Database | Neon or Supabase | Managed with point-in-time recovery; branching gives production-shaped previews cheaply |
| Cache and queue | Upstash | Managed, encrypted, usage-priced |
| Object storage | Cloudflare R2 | Egress-free, standard protocol, interchangeable |
| Observability | Grafana Cloud and Sentry | Standard protocols; the backend is replaceable |

**No hosting-vendor-proprietary runtime interface appears in application code.** Everything runs as a plain process and a container. This topology is a cost-and-convenience preference, not a dependency.

---

## L.5 Deployment process

```
   change proposed
        ↓
   pipeline runs in full (Part L.3)
        ↓
   preview environment provisioned with a branched database
        ↓
   review and approval
        ↓
   merge
        ↓
   MIGRATIONS APPLY FIRST — expand phase only
        ↓
   staging deploys automatically
        ↓
   staging smoke suite
        ↓
   manual approval
        ↓
   production deploys — rolling
        ↓
   post-deploy verification
        ↓
   fifteen-minute watch on error rate and latency
        ↓
   automatic rollback on regression
```

### Zero-downtime mechanics

| Tier | Mechanism |
|---|---|
| Web | Atomic platform deploy with instant rollback; versions briefly coexist, which the migration discipline accounts for |
| Worker | Rolling restart, one instance at a time: stop accepting jobs, finish in-flight work within a grace period, exit. Unfinished jobs return to the queue and are safe to retry by construction. |
| Scheduler | Old leader stops, lock releases, new leader acquires within seconds. All scheduled jobs are idempotent, so a missed or duplicated tick is harmless. |
| Long-running steps | Steps exceeding the grace period are detected by stale heartbeat and requeued by the recovery task |

### Migration discipline

**Expand and contract, always.** Every schema change must leave the previously deployed application version working:

```
   Release N     add a nullable column
   Release N     backfill as a monitored job — never inside the migration
   Release N+1   write both old and new
   Release N+2   read new only
   Release N+3   remove the old column
```

| Rule | Reason |
|---|---|
| Migrations apply **before** the new application version deploys | The new schema must be ready when new code arrives |
| Indexes created without blocking | Avoids locking a table under read load |
| Lock and statement timeouts on all schema changes | A migration that would block fails fast rather than taking the application down |
| Backfills are jobs | Resumable, rate-limited, observable |
| Every migration tested against production-shaped data with timing recorded | Slow migrations are identified before they reach production |
| Rollback documented per migration | Either a tested down script, or an explicit forward-only note with its compensating action |

**Feature flags decouple deployment from release.** Every new engine ships disabled and is enabled deliberately. Flags are removed within two releases of full rollout so they do not accumulate.

---

## L.6 Environment variables

Validated at process start against a schema. **Invalid configuration prevents start-up** rather than producing a runtime failure at first use.

| Group | Contents |
|---|---|
| **Runtime** | Environment name, application address, log level |
| **Database** | Primary connection, optional replica connection, pool sizes |
| **Cache and queue** | Connection, per-queue concurrency limits |
| **Storage** | Endpoint, bucket names, credentials, signing lifetimes |
| **Encryption** | Key management identifier, current key version |
| **AI** | Credential, **tier-to-model binding map**, per-tier concurrency and token budgets |
| **Image generation** | Credential, concurrency limit, default variant count |
| **Marketplace** | Application identifier, redirect address, rate limits, reserved quota share |
| **Fulfilment** | Rate limits, catalogue sync scope |
| **Trademark registries** | Endpoints, credentials, cache lifetime |
| **Sessions** | Signing secret, idle and absolute expiry |
| **Budgets** | Default per-run budget, monthly cap, behaviour at cap |
| **Limits** | Concurrent runs, per-shop listing cap, per-run listing cap |
| **Observability** | Trace collector endpoint, error reporting credential, sample rates |
| **Behaviour flags** | Provider mocking, feature flag defaults |

### Rules

| Rule | Detail |
|---|---|
| Secrets from the platform store only | Never in source, never in images, never in the repository |
| Provider mocking rejected in production | The combination is invalid and prevents start-up |
| **Model bindings are configuration** | A model upgrade is a configuration change plus an evaluation run, not a code change |
| Budgets are configuration | Adjustable without deployment |
| No default that would spend money | Any missing spending-related value fails start-up rather than assuming a default |

---

## L.7 Monitoring

### Signals

| Signal | Retention |
|---|---|
| Structured logs with mandatory context fields | Weeks hot, longer for audit-relevant events |
| Distributed traces spanning web, queue, worker and every external call | Days |
| Metrics — request rates, queue depth and age, provider latency and errors, step durations | Long |
| Errors with source mapping, release tagging and workspace context — never secrets or personal data | Months |
| **Cost telemetry — tokens, images, provider calls, currency per run and per step** | Indefinite |
| External uptime checks from multiple regions | Long |

**Every log line carries:** timestamp, level, service, environment, trace identifier, workspace identifier, run identifier, step key, event and duration. A redaction list strips credentials and personal data before anything is written.

**A run's entire execution is viewable as a single trace** — across the web tier, the queue, multiple workers and every external call — because trace context propagates through job payloads.

### Dashboards, defined as code

System health · pipeline throughput and failure reasons · **cost against budget** · provider health including quota and breaker state · business outcomes including products published and prediction accuracy · queue depth, age and dead letters.

### Alerts

| Alert | Severity |
|---|---|
| API error rate elevated | High |
| Run failure rate elevated | High |
| Queue age beyond threshold | High |
| Provider circuit breaker open for a sustained period | High |
| **Publishing failure** | **Critical** |
| **Legal engine failure** | **Critical** |
| Marketplace credentials nearing expiry | High |
| Budget consumption approaching the cap | High |
| Database connections approaching saturation | High |
| Replica lag elevated | Medium |
| Storage growth above trend | Medium |
| Cost anomaly on a single run | Medium |
| Dead letter entries appearing | High |
| Audit chain discontinuity | Critical |

**Every alert links to a runbook. An alert without a runbook is a defect**, not an acceptable state.

---

## L.8 Logging

| Concern | Approach |
|---|---|
| Format | Structured, machine-parseable |
| Context propagation | Bound automatically to the execution context — never passed manually through call chains |
| Redaction | Credentials, tokens and personal data stripped before writing, by an allow-list not a deny-list |
| Levels | Debug locally, informational in production, with per-module overrides available without redeployment |
| **Correlation** | **Every user-visible error carries an identifier that ties it to its server-side trace** |
| Sampling | High-volume informational logs sampled; all warnings and errors retained |
| Audit separation | Audit events are database rows, not log lines — logs are for diagnosis, the audit log is a record |

---

## L.9 Backups and disaster recovery

| Asset | Method | Recovery point | Recovery time |
|---|---|---|---|
| Database | Continuous archiving with point-in-time recovery, plus a nightly logical dump to separate storage | Minutes | About an hour |
| Object storage | Versioning plus cross-region replication | Near zero | Under an hour |
| Cache and queue | **Ephemeral by design.** No durable state. Incomplete runs rebuild from step records. | Not applicable | Minutes |
| Secrets | Platform-managed with versioning, plus encrypted offline escrow for emergency access | Not applicable | About an hour |
| Infrastructure definitions | Version-controlled with remote state versioning | Near zero | Minutes |

### Recovery scenarios

| Scenario | Procedure |
|---|---|
| Accidental deletion | Restore from the soft-delete recovery window — minutes |
| Bad migration | Roll back the application; forward-fix the schema. Point-in-time restore only if data was destroyed. |
| Database loss | Point-in-time restore to a new instance, repoint, rebuild queue state from step records |
| Region outage | Restore into a secondary region from replicated backups; redirect |
| Credential compromise | Kill switches, revoke all tokens, invalidate sessions, rotate, audit |
| Provider permanently unavailable | Adapter substitution behind the existing interface — days, not a rewrite |

**Restore drills run monthly and the result is recorded.** A backup that has never been restored is a hypothesis, not a backup. This is the single most commonly skipped operational practice and the one most likely to be discovered missing at the worst moment.

---

## L.10 Runbooks

Every runbook follows one template: **symptom · detection signal · immediate mitigation · diagnosis · resolution · verification · prevention follow-up.**

| Runbook | Covers |
|---|---|
| Stuck run | Steps with stale heartbeats, recovery task behaviour, manual requeue |
| Queue backlog | Distinguishing depth from age; when adding workers is wrong |
| Provider outage | Breaker state, operator communication, resumption |
| Budget exhausted | Raising limits, identifying the runaway step |
| Credential expiry or revocation | Reconnection, resuming paused operations |
| Duplicate listing | Detection, reconciliation, removal, root cause |
| Bad scoring configuration | Reverting, re-scoring |
| Database failover | Promotion, reconnection, verification |
| Credential compromise | Kill switches, rotation, audit, notification |
| Cost spike | Identification, throttling, retrospective |
| Restore drill | Monthly verification procedure |
| Marketplace rate exhaustion | Reserved quota behaviour, prioritisation |

---

## L.11 Release management

| Practice | Detail |
|---|---|
| Versioning | Semantic, with every deployment tagged and linked to its commit range and migration set |
| Release notes | Generated from commit history, reviewed for accuracy |
| Feature flags | Every new engine ships disabled; removed within two releases of full rollout |
| Application rollback | One action, platform-native |
| **Schema rollback** | **Deliberately not the default path** — expand-and-contract means the previous application version keeps working against the new schema |
| Post-deploy verification | Automated smoke suite: authenticate, run a mocked pipeline, load a report, create a mocked draft |
| Watch period | Fifteen minutes on error rate and latency, with automatic rollback on regression |
| Change freeze | None while single-user. At multi-user scale, freeze during high-traffic commercial periods. |

---

## L.12 Operational readiness checklist

Before any phase is considered deployed:

- [ ] All environment variables validated at start-up
- [ ] Migrations tested against production-shaped data with timing recorded
- [ ] Rolling restart verified to drain in-flight work without loss
- [ ] Dashboards created for every new component
- [ ] Alerts configured with runbooks linked
- [ ] Cost telemetry emitting for every new spending path
- [ ] Backup and restore drilled since the last significant schema change
- [ ] Feature flags in place for anything new
- [ ] Rollback path documented and tested
- [ ] Kill switches verified
- [ ] Chaos test passing: worker killed mid-run produces identical output with zero duplicate effects
- [ ] Isolation suite passing on every endpoint

---

# Summary of Phase 1B decisions

| Area | Decision |
|---|---|
| **Architecture style** | Modular monolith with a separate worker tier and a singleton scheduler |
| **Frontend** | Server-first React with a typed API, virtualised tables, progress via a resumable event stream |
| **Backend** | Single language across both tiers so scoring and validation logic is shared, not duplicated |
| **Layering** | A pure, input-output-free domain package containing all arithmetic, enforced by build-time dependency analysis |
| **Database** | One relational database serving relational, time-series, search and vector roles; partitioned from day one |
| **Data access** | ORM behind a repository layer that is the only place it is called |
| **Tenancy** | Workspace identifier on every table from the first migration; isolation policies written now, enforced later |
| **Observations** | Immutable, dated snapshots separated from stable identity records |
| **Scoring** | Deterministic arithmetic with versioned weights stored as data, never produced by a model |
| **Orchestration** | Durable, resumable step state machine in the database; the queue carries delivery, not truth |
| **AI** | Capability tiers rather than model names; structured output contracts; versioned prompt registry; aggregation before prompting |
| **Injection defence** | Structural — external text never reaches the generative stage |
| **Integrations** | Uniform adapter contract; no vendor library outside its adapter |
| **Market data** | A provider chain whose default path requires no automated access to anyone's systems and therefore cannot break |
| **Legal gate** | Enforced in the service layer where the external call is made, not in the interface |
| **Publishing** | Decomposed into eight idempotent operations; duplicate prevention at four layers; draft-only automation with no code path to direct activation |
| **Security** | Envelope encryption with per-workspace keys; mandatory second factor; step-up authentication for anything irreversible |
| **Scaling** | External rate limits and AI cost bite before CPU or database throughput — the plan is ordered accordingly |
| **Deployment** | Three processes, expand-and-contract migrations, zero-downtime rolling restarts, monthly restore drills |

---

## Open questions carried into Phase 2

These require answers before implementation of the affected areas begins.

| # | Question | Blocks |
|---|---|---|
| 1 | What is the realistic, compliant path to sales and revenue estimate data? Is a partner or interface arrangement obtainable? | Market data adapter priority |
| 2 | What minimum sample sizes and significance thresholds should govern the suppression of findings? | Statistical layer |
| 3 | Which registries and goods classes constitute adequate legal coverage, and what default risk appetite should be encoded? | Legal engine |
| 4 | Exact print dimensions, resolutions and placement rules for each of the six supported products | Artwork and placement |
| 5 | Confirmation of every marketplace fee component and its treatment | Pricing engine |
| 6 | What outcome data is reliably retrievable for the operator's own listings, and at what frequency? | Performance sync and learning loop |
| 7 | Which fulfilment providers are pre-approved, affecting the catalogue whitelist? | Catalogue sync scope |
| 8 | Target minimum margin, and whether the floor blocks or warns | Pricing enforcement |

---

## Document control

| | |
|---|---|
| **Phase** | 1B of the POD Intelligence specification series |
| **Covers** | System architecture, technology decisions, database design, API design, backend services, AI architecture, integration architecture, background processing, security, scalability, project structure, deployment |
| **Excludes** | Implementation. No application code. |
| **Prerequisite** | [Part 1 — Product Definition](PART-1-product-definition.md) |
| **Next** | Phase 2 — implementation, beginning with foundation and orchestration |
| **Status** | Ready for engineering review |
