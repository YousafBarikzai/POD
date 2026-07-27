# 08 — API Architecture

**Version:** 1.0

Two surfaces, one domain layer:

| Surface | Consumer | Phase | Transport |
|---|---|---|---|
| **Internal API** (`tRPC v11`) | The Next.js web client only | Phase 1 | HTTP POST/GET over `/api/trpc`, SSE for streams |
| **Public API** (`REST /api/v1`) | Third parties, integrations, future mobile | Phase 6 (SaaS) | REST + OpenAPI 3.1, API-key or OAuth |

**Rule:** both surfaces are thin. All logic lives in `packages/domain` and `apps/api/src/services`. A REST handler and a tRPC procedure calling the same use case must be ≤ 10 lines each. This is what makes adding the public API a Phase-6 task rather than a rewrite.

---

## 1. Why tRPC first

- Single-consumer, single-repo, single-team: the cost of an IDL (OpenAPI codegen) buys nothing yet.
- End-to-end type safety eliminates an entire class of integration bugs on a data-heavy UI with ~60 distinct payload shapes.
- Zod schemas at the procedure boundary double as runtime validation *and* as the source for the REST OpenAPI spec later (`trpc-to-openapi` or hand-written façade — decision deferred to Phase 6; the domain layer is unaffected either way).

Recorded as [ADR-0011](adr/ADR-0011-trpc-internal-rest-public.md).

---

## 2. Router structure

```
appRouter
├── auth          login, logout, session, totpEnrol, totpVerify, recoveryCodes
├── workspace     get, update, settings, members(SaaS), usage
├── niche         list, get, create, update, reject, search, stats
├── run           create, get, list, cancel, pause, resume, retryStep, raiseBudget,
│                 clone, diff, progress(SSE), estimate
├── report        opportunity, competitors, competitorListings, success, failure,
│                 gaps, coverage, export
├── concept       listByRun, get, select, deselect, regenerateOne, regenerateAll,
│                 createManual, expand, score, delete
├── legal         screenConcept, screenArtwork, screenListingText, get, override,
│                 acceptAlternative, blocklist
├── artwork       generate, get, listByConcept, accept, reject, removeBackground,
│                 upscale, vectorise, autocrop, qa, originality, upload, download
├── product       recommendations, createDraft, getDraft, listDrafts, updateDraft,
│                 selectVariants, price, checklist
├── seo           generate, list, get, regenerateOne, update, select, validate
├── printify      catalogueSync, uploadArtwork, createProduct, mockups, status
├── etsy          connect, callback, disconnect, createDraft, uploadImages,
│                 setInventory, publish, syncListing, shopInfo
├── listing       list, get, performance, resync, lineage
├── analytics     portfolio, performance, accuracy, costs, niches, throughput
├── learning      configs, activeConfig, proposals, backtest, activate, reject, rescore
├── integration   list, health, connect, disconnect, test, quota
├── asset         signedUrl, upload, delete
├── search        global
├── notification  list, markRead, markAllRead, preferences
└── admin         featureFlags, dlq, consistencyReport   (owner role only)
```

### 2.1 Procedure kinds

| Kind | Base | Guarantees |
|---|---|---|
| `publicProcedure` | — | Rate-limited by IP; auth endpoints only |
| `protectedProcedure` | session required | Injects `ctx.user`, `ctx.workspaceId`; sets `app.workspace_id` on the DB connection |
| `writeProcedure` | protected + CSRF + audit | Wraps in a transaction, writes `audit_log` |
| `spendProcedure` | write + budget check | Rejects with `BUDGET_EXCEEDED` before any external call |
| `idempotentProcedure` | write + `idempotency-key` header | Replays the stored response for a repeated key within 24 h |
| `adminProcedure` | protected + role `owner` | |

### 2.2 Middleware chain

```
requestId → tracing(OTel span) → rateLimit → auth → workspaceContext →
tenantDbSession → csrf(mutations) → inputValidation(Zod) → idempotency →
budgetGuard → audit → handler → outputValidation → errorMapper → metrics
```

---

## 3. Representative contracts

Written as TypeScript for the internal API; the REST equivalents are mechanical.

### 3.1 `run.create`

```ts
input: z.object({
  nicheId: z.string().uuid().optional(),
  nicheName: z.string().min(2).max(80).optional(),
  productTypeId: z.string().uuid(),
  style: z.enum(['vintage','typography','hand_drawn','illustration','humour','modern','auto'])
          .default('auto'),
  depth: z.enum(['quick','standard','deep']).default('standard'),
  budgetAmount: z.number().int().positive().optional(),
  seedKeywords: z.array(z.string().max(40)).max(20).default([]),
  excludedTerms: z.array(z.string().max(40)).max(20).default([]),
}).refine(v => v.nicheId || v.nicheName, 'nicheId or nicheName required')

output: z.object({
  runId: z.string().uuid(),
  status: RunStatus,
  steps: z.array(z.object({ key: z.string(), order: z.number(), status: StepStatus })),
  estimate: z.object({
    durationSecondsP50: z.number(),
    costAmount: z.number(), costCurrency: z.string(),
  }),
})
```
Headers: `Idempotency-Key` (required). Errors: `BUDGET_EXCEEDED`, `INTEGRATION_NOT_CONNECTED`, `CONCURRENCY_LIMIT`, `VALIDATION_ERROR`.

### 3.2 `run.progress` (SSE subscription)

```ts
input: z.object({ runId: z.string().uuid(), lastEventId: z.string().optional() })

// Event stream
type RunProgressEvent =
  | { type: 'step.started';   stepKey: string; at: string }
  | { type: 'step.progress';  stepKey: string; pct: number; message: string; data?: unknown }
  | { type: 'step.completed'; stepKey: string; durationMs: number; costAmount: number;
      outputRef: { table: string; id: string } }
  | { type: 'step.failed';    stepKey: string; error: ApiError; retriable: boolean }
  | { type: 'run.status';     status: RunStatus; spendAmount: number; budgetAmount: number }
  | { type: 'partial.result'; kind: 'opportunity'|'subniches'|'shops'|'factors'; payload: unknown }
  | { type: 'heartbeat';      at: string }
```

Transport: `text/event-stream`, `id:` on every event for `Last-Event-ID` resumption, heartbeat every 15 s, server-side backpressure by coalescing `step.progress` events to at most 2/second per step. Client falls back to polling `run.get` every 3 s if SSE fails twice.

### 3.3 `concept.select`

```ts
input:  z.object({ runId: z.string().uuid(), conceptIds: z.array(z.string().uuid()).min(1).max(20) })
output: z.object({
  selected: z.number(),
  screeningJobIds: z.array(z.string().uuid()),
  estimatedArtworkCostAmount: z.number(),
})
```
Side effect: enqueues legal screening for each concept. Idempotent by `(runId, sorted conceptIds)`.

### 3.4 `artwork.generate`

```ts
input: z.object({
  conceptId: z.string().uuid(),
  briefId: z.string().uuid().optional(),      // defaults to latest
  variantCount: z.number().int().min(1).max(8).default(4),
  steer: z.string().max(500).optional(),
})
output: z.object({ artworkIds: z.array(z.string().uuid()), estimatedCostAmount: z.number() })
```
**Guard (FR-901):** the service asserts the concept's latest `legal_screening` is `none`/`low`, or `medium`/`high` with a recorded override. Otherwise `409 LEGAL_GATE_NOT_PASSED` — and no provider call is made.

### 3.5 `product.price`

```ts
input: z.object({
  draftId: z.string().uuid(),
  mode: z.enum(['target_margin','fixed_price']),
  targetMarginPct: z.number().min(0).max(0.9).optional(),
  retailPriceAmount: z.number().int().positive().optional(),
  freeShipping: z.boolean().default(true),
})
output: z.object({
  retailPriceAmount: z.number(), currency: z.string(),
  breakdown: z.array(z.object({
    key: z.enum(['production','listing_fee','transaction_fee','payment_fee',
                 'offsite_ads','shipping','vat','net_profit']),
    label: z.string(), amount: z.number(), pctOfPrice: z.number(),
  })),
  netProfitAmount: z.number(), marginPct: z.number(),
  meetsFloor: z.boolean(), floorPct: z.number(),
  minimumPriceForFloorAmount: z.number(),
  competitorContext: z.object({ p25: z.number(), median: z.number(), p75: z.number() }),
})
```

### 3.6 `etsy.publish`

```ts
input:  z.object({ draftId: z.string().uuid(), confirmation: z.literal('PUBLISH') })
output: z.object({
  listingId: z.string().uuid(), etsyListingId: z.number(),
  url: z.string().url(), state: z.literal('active'), publishedAt: z.string(),
})
```
`spendProcedure` + `idempotentProcedure`. Pre-flight re-evaluates the full hard checklist server-side; the UI's disabled button is a convenience, not the control. Errors: `CHECKLIST_FAILED` (with the failing items), `ETSY_VALIDATION_ERROR` (field-mapped), `NEEDS_REAUTH`, `RATE_LIMITED`.

### 3.7 `report.competitorListings` (cursor pagination)

```ts
input: z.object({
  runId: z.string().uuid(),
  cursor: z.string().optional(),
  limit: z.number().int().min(10).max(200).default(50),
  sort: z.enum(['est_monthly_sales','price','reviews','age','image_count']).default('est_monthly_sales'),
  direction: z.enum(['asc','desc']).default('desc'),
  filters: z.object({
    shopIds: z.array(z.string().uuid()).optional(),
    paletteFamily: z.array(PaletteFamily).optional(),
    typography: z.array(TypographyStyle).optional(),
    priceMin: z.number().optional(), priceMax: z.number().optional(),
    cohort: z.enum(['top_decile','top_quartile','middle','bottom_quartile','bottom_decile']).optional(),
  }).default({}),
})
output: z.object({
  items: z.array(CompetitorListingRow),
  nextCursor: z.string().nullable(),
  totalEstimate: z.number(),          // approximate; exact counts are not promised on large sets
  facets: z.record(z.string(), z.record(z.string(), z.number())),
})
```

Cursor encoding: base64url of `{sortValue, id}` — keyset pagination, stable under concurrent inserts. Offset pagination is prohibited (NFR-S5).

---

## 4. Error model

A single error shape across both surfaces.

```ts
type ApiError = {
  code: ErrorCode          // stable machine-readable enum
  message: string          // human, safe to display
  detail?: string          // additional context, safe to display
  field?: string           // for validation errors, maps to a UI field
  retriable: boolean
  retryAfterSeconds?: number
  correlationId: string
  docsUrl?: string
}
```

| Code | HTTP | Meaning |
|---|---|---|
| `VALIDATION_ERROR` | 400 | Input failed schema or business validation |
| `UNAUTHENTICATED` | 401 | No/invalid session |
| `FORBIDDEN` | 403 | Authenticated but not permitted |
| `NOT_FOUND` | 404 | Entity absent or not in this workspace (indistinguishable by design) |
| `CONFLICT` | 409 | State conflict (e.g. already published) |
| `LEGAL_GATE_NOT_PASSED` | 409 | Artwork attempted before clearance |
| `CHECKLIST_FAILED` | 409 | Publish attempted with failing hard checks |
| `BUDGET_EXCEEDED` | 402 | Run or workspace budget exhausted |
| `CONCURRENCY_LIMIT` | 429 | Too many active runs |
| `RATE_LIMITED` | 429 | Client or provider quota |
| `INTEGRATION_NOT_CONNECTED` | 424 | Required provider missing |
| `NEEDS_REAUTH` | 424 | Token revoked/expired |
| `PROVIDER_UNAVAILABLE` | 503 | Circuit breaker open |
| `PROVIDER_ERROR` | 502 | Upstream failure, mapped |
| `AI_SCHEMA_INVALID` | 502 | Model output failed validation after repair |
| `INTERNAL_ERROR` | 500 | Unexpected |

**Rules:** raw provider messages are never surfaced as `message`; they are mapped through `packages/adapters/*/errors.ts` and the raw text is logged only. `NOT_FOUND` is returned for cross-tenant access attempts so the API never confirms the existence of another workspace's data.

---

## 5. Idempotency

| Aspect | Rule |
|---|---|
| Header | `Idempotency-Key: <uuid>` required on `run.create`, `artwork.generate`, all `printify.*` and `etsy.*` mutations, and `product.createDraft` |
| Storage | `idempotency_records(key, workspace_id, request_hash, response_body, status, created_at)` with a 24-hour TTL |
| Semantics | Same key + same request hash → replay the stored response. Same key + different hash → `409 IDEMPOTENCY_KEY_REUSE` |
| In-flight | A second request with the same key while the first is running returns `409 REQUEST_IN_PROGRESS` with `Retry-After` |
| Internal | Every side-effecting external call also carries its own derived idempotency key (`sha256(draftId:operation:attemptInputHash)`) stored on `publish_jobs`, so retries at the queue level cannot duplicate a listing |

---

## 6. Rate limiting

| Scope | Limit | Store |
|---|---|---|
| Unauthenticated (login, callback) | 10/min per IP, 30/hour | Redis sliding window |
| Authenticated reads | 600/min per user | Redis token bucket |
| Authenticated writes | 120/min per user | Redis token bucket |
| `spendProcedure` | 30/min per workspace | Redis |
| Concurrent runs | 3 per workspace (Phase 1) | Redis counter with lease |
| Outbound Etsy | 10/s, 10,000/day per integration, 15% reserved for publish | Redis token bucket, shared across processes |
| Outbound Printify | per documented limit, adaptive on 429 | Redis token bucket |
| Outbound Ideogram | configurable concurrency (default 4) | Redis semaphore |
| Outbound AI | tier-specific concurrency + tokens/min | Redis token bucket |

Responses include `RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset`, and `Retry-After` on 429.

---

## 7. Webhooks

### 7.1 Inbound

| Source | Endpoint | Purpose | Verification |
|---|---|---|---|
| Printify | `POST /api/webhooks/printify` | product publish status, mockup ready, order events | HMAC-SHA256 signature header, timestamp within 5 min, replay cache on event id |
| Etsy | `POST /api/webhooks/etsy` | listing state changes (where supported) | Signature + shop-id match |
| Stripe (SaaS) | `POST /api/webhooks/stripe` | subscription lifecycle | Stripe signature |

**Handling contract:** verify → persist raw event → return 200 within 500 ms → process asynchronously via queue. Never do work in the request. Duplicate event ids are acknowledged and dropped.

### 7.2 Outbound (Phase 6)

Workspace-configurable subscriptions to: `run.completed`, `run.failed`, `concepts.ready`, `artwork.ready`, `draft.created`, `listing.published`, `listing.first_sale`, `budget.threshold`, `legal.blocked`.

Delivery: HMAC-signed POST, exponential retry (1m, 5m, 30m, 2h, 12h), dead-lettered after 5 failures, per-endpoint circuit breaker, delivery log visible in settings.

---

## 8. Public REST API (Phase 6 design)

Base: `https://api.podintelligence.app/v1` · Auth: `Authorization: Bearer <api_key>` or OAuth 2.0 client credentials · Format: JSON, `application/json` · Spec: OpenAPI 3.1 generated in CI and published.

```
POST   /v1/runs                          create a research run
GET    /v1/runs/{id}                     run status + step states
GET    /v1/runs/{id}/events               SSE progress
GET    /v1/runs/{id}/opportunity          opportunity report
GET    /v1/runs/{id}/competitors          paginated competitor listings
GET    /v1/runs/{id}/success              success factors
GET    /v1/runs/{id}/failure              anti-factors
GET    /v1/runs/{id}/gaps                 ranked gaps
GET    /v1/runs/{id}/concepts             concepts
POST   /v1/concepts/{id}/select
POST   /v1/concepts/{id}/artwork          generate artwork
GET    /v1/artworks/{id}
POST   /v1/products                       create draft
POST   /v1/products/{id}/seo              generate SEO
POST   /v1/products/{id}/publish
GET    /v1/listings                       paginated
GET    /v1/listings/{id}/performance
GET    /v1/usage                          metered usage for the period
```

**Conventions:** cursor pagination via `?cursor=&limit=`; `Idempotency-Key` on all POSTs; RFC 9457 `application/problem+json` for errors (mapping the internal `ApiError` one-to-one); versioning by URL path, with a minimum 6-month deprecation window announced via `Sunset` and `Deprecation` headers.

---

## 9. Caching

| Layer | What | TTL / invalidation |
|---|---|---|
| HTTP | Static assets, fonts | Immutable, 1 year, content-hashed |
| CDN | Signed asset URLs (thumbnails, mockups) | 1 hour signed, edge-cached |
| React Query | All read queries | `staleTime` 30 s default; 5 min for reports (immutable); 0 for run status |
| Redis | Blueprint catalogue, print providers, taxonomy | 24 h, invalidated by sync job |
| Redis | Trademark registry lookups | 30 days by normalised term |
| Redis | Provider quota counters | rolling |
| Postgres | Vision analysis by `image_hash` | permanent (keyed by extractor version) |
| Postgres | AI responses by `input_hash` | permanent for deterministic-purpose calls (classification), never for creative calls |

**Invalidation discipline:** every mutation declares the query keys it invalidates in a single map (`packages/api/src/cache-map.ts`) so cache correctness is reviewable in one file rather than scattered across handlers.

---

## 10. API security

| Control | Implementation |
|---|---|
| Transport | TLS 1.3 only, HSTS preload |
| Session | `HttpOnly`/`Secure`/`SameSite=Lax` cookie, opaque token, hash stored server-side |
| CSRF | Double-submit token on all cookie-authenticated mutations; SSE endpoints are GET and read-only |
| Authorisation | Workspace scope injected server-side from the session — never accepted from client input. Any procedure that takes an entity id resolves and re-checks its `workspace_id`. |
| Input | Zod at the boundary; explicit max lengths and array sizes on every field |
| Output | Response schemas validated in development and in CI; serialisers strip internal fields (`*_enc`, raw provider bodies, prompt internals) |
| Injection | Parameterised queries only (Prisma); no string-built SQL; `jsonb` paths validated |
| Headers | CSP with per-request nonce, `X-Content-Type-Options`, `Referrer-Policy: strict-origin-when-cross-origin`, `Permissions-Policy` minimal |
| Uploads | Type sniffing by magic bytes, size caps, stored outside the web root, served via signed URLs only, never executed |
| Enumeration | Cross-tenant reads return `NOT_FOUND`, timing-normalised |

---

## 11. Versioning & compatibility

- **Internal tRPC:** versionless; client and server deploy together. Breaking changes are permitted within a release but a two-release deprecation window applies to any procedure the client may cache against.
- **Public REST:** path-versioned. Additive changes (new optional fields, new endpoints) are non-breaking and shipped freely. Breaking changes require `v2` and a 6-month sunset.
- **Webhooks:** payloads carry `schema_version`; consumers must tolerate unknown fields.
- **Contract tests:** the OpenAPI spec is generated in CI and diffed; a breaking diff fails the build unless the PR carries a `breaking-api` label and an ADR.
