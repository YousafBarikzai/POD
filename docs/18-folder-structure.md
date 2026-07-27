# 18 — Folder Structure & Code Organisation

**Version:** 1.0 · pnpm workspaces + Turborepo

---

## 1. Top level

```
pod-intelligence/
├── apps/
│   ├── web/                  Next.js 15 — UI + tRPC handler + webhooks
│   ├── worker/               Node — queue consumers, pipeline execution
│   └── scheduler/            Node — cron leader
├── packages/
│   ├── contracts/            Shared Zod schemas, enums, branded types
│   ├── domain/               Pure logic: scoring, stats, pricing, validators
│   ├── db/                   Prisma schema, client, repositories, migrations
│   ├── queue/                Queue abstraction, job registry, DLQ
│   ├── orchestrator/         Run DAG, step runner, budget guard, resume
│   ├── engines/              One package per pipeline engine
│   ├── adapters/             One package per external provider
│   ├── observability/        Logging, tracing, metrics, cost telemetry
│   ├── ui/                   Shared React components + design tokens
│   └── config/               Env schema, feature flags, model map
├── docs/                     This specification set
├── infra/                    Terraform, Docker, scripts
├── tooling/                  ESLint, TS configs, Prettier, dep-cruiser rules
├── e2e/                      Playwright suites
├── turbo.json
├── pnpm-workspace.yaml
└── package.json
```

**Why a monorepo:** the domain logic is shared by the web tier, the worker and the future public API. Splitting repositories would force premature package publishing and version skew between a UI and a pipeline that must stay in lockstep.

---

## 2. `apps/web`

```
apps/web/
├── src/
│   ├── app/                                Next.js App Router
│   │   ├── (auth)/login/ · totp/ · recover/
│   │   ├── (app)/
│   │   │   ├── layout.tsx                  shell: sidebar, topbar, providers
│   │   │   ├── page.tsx                    dashboard
│   │   │   ├── research/
│   │   │   │   ├── page.tsx                run history
│   │   │   │   ├── new/page.tsx            wizard
│   │   │   │   └── [runId]/
│   │   │   │       ├── page.tsx            live monitor
│   │   │   │       ├── opportunity/ · competitors/ · success/
│   │   │   │       ├── failure/ · gaps/ · concepts/
│   │   │   │       └── compare/[otherRunId]/
│   │   │   ├── concepts/[conceptId]/
│   │   │   ├── artwork/[artworkId]/
│   │   │   ├── products/[draftId]/         tabs + review/
│   │   │   ├── listings/[listingId]/
│   │   │   ├── analytics/                  performance · accuracy · costs · learning
│   │   │   ├── niches/
│   │   │   └── settings/                   account · workspace · integrations · economics
│   │   │                                   · scoring · data · notifications · budgets
│   │   └── api/
│   │       ├── trpc/[trpc]/route.ts
│   │       ├── sse/runs/[runId]/route.ts
│   │       ├── webhooks/printify/ · etsy/ · stripe/
│   │       ├── auth/etsy/callback/route.ts
│   │       └── health/route.ts
│   ├── server/
│   │   ├── trpc/
│   │   │   ├── root.ts                     appRouter composition
│   │   │   ├── trpc.ts                     procedure builders + middleware
│   │   │   ├── context.ts
│   │   │   └── routers/                    one file per router (doc 8 §2)
│   │   ├── services/                       use-case orchestration (thin)
│   │   ├── auth/                           session, TOTP, password
│   │   └── cache-map.ts                    mutation → invalidated query keys
│   ├── components/
│   │   ├── charts/ · tables/ · scores/ · runs/ · concepts/ · artwork/
│   │   ├── products/ · reports/ · layout/ · forms/
│   ├── hooks/ · lib/ · stores/ · styles/ · messages/en-GB.json
└── tests/
```

**Conventions**
- Server Components by default; `'use client'` only where interactivity demands it, pushed to the leaves.
- Route-level `loading.tsx` and `error.tsx` everywhere; report sections wrapped in independent Suspense boundaries.
- Data fetching in Server Components for the initial payload, TanStack Query for anything interactive or streaming.
- No business logic in components — components render; `server/services` orchestrate; `packages/domain` computes.
- All user-facing strings from `messages/en-GB.json` (NFR-I1).

---

## 3. `apps/worker` and `apps/scheduler`

```
apps/worker/src/
├── index.ts               boot: config, telemetry, queues, graceful shutdown
├── health.ts              healthcheck endpoint for the container
├── processors/
│   ├── run.start.ts · run.step.ts
│   ├── vision.extract.ts · embedding.compute.ts
│   ├── artwork.generate.ts · artwork.process.ts
│   ├── publish.printify.ts · publish.etsy.ts
│   ├── sync.performance.ts · sync.catalogue.ts
│   ├── learning.recalibrate.ts
│   └── maintenance.*.ts   reaper · retention · consistency · partitions · search
├── lifecycle/             signal handling, drain, lease management
└── registry.ts            queue → processor binding

apps/scheduler/src/
├── index.ts               Redis leader lock, tick loop
└── jobs.ts                cron definitions (all idempotent)
```

---

## 4. `packages/domain` — the crown jewels

Pure, synchronous, dependency-free (except `contracts`). Every function here is exhaustively unit-tested, including property-based tests. **No I/O, no dates from `Date.now()`, no randomness without an injected source.**

```
packages/domain/src/
├── scoring/
│   ├── opportunity.ts        five sub-scores + overall + verdict banding
│   ├── shop-selection.ts     selection score incl. the <3-year age preference
│   ├── design-success.ts     five predictor dimensions + contribution vector
│   ├── gap.ts                gap opportunity score + demand floor
│   ├── product.ts            demand/competition/profitability for configurations
│   ├── seo-quality.ts        variation quality scoring
│   ├── normalise.ts          min-max, z-score, winsorise, log-scale
│   └── bands.ts              score → band mapping
├── stats/
│   ├── cohorts.ts            age-normalised percentile cohorting
│   ├── contingency.ts        support, lift, two-proportion z-test, χ²
│   ├── effect-size.ts        Cliff's delta, Cohen's h
│   ├── correlation.ts        Spearman, multicollinearity detection
│   ├── bands.ts              optimal numeric band search
│   ├── interactions.ts       pairwise interaction lift
│   └── confidence.ts         n + p → low/medium/high
├── pricing/
│   ├── fee-stack.ts          Etsy fee model, both offsite-ads cases
│   ├── solve.ts              target-margin ↔ fixed-price solve
│   ├── margin.ts             floor evaluation
│   └── money.ts              minor-unit arithmetic, rounding, currency
├── text/
│   ├── normalise.ts · ngrams.ts · tfidf.ts · similarity.ts
│   └── validators.ts         Etsy title/tag/description rules
├── colour/
│   ├── lab.ts · kmeans.ts · palette-family.ts · gamut.ts
├── placement/                per-product-type artwork placement maths
├── qa/                       print-readiness criteria evaluation
├── legal/
│   └── risk-rules.ts         the deterministic risk-level rule table
└── learning/
    ├── features.ts · shrinkage.ts · backtest-metrics.ts
```

**This package is the product's intellectual property in code form.** It is why scores are reproducible, why the learning loop is auditable, and why the whole system can be tested without a network.

---

## 5. `packages/engines`

```
packages/engines/
├── opportunity/ ├── competitor/ ├── style-extraction/
├── success/     ├── failure/    ├── gap/
├── predictor/   ├── concept/    ├── legal/
├── artwork/     ├── product/    ├── seo/
├── publishing/  └── learning/
```

Each follows an identical shape:

```
<engine>/src/
├── index.ts          export execute(ctx: StepContext): Promise<StepOutput>
├── execute.ts        orchestration: fetch → compute → persist
├── inputs.ts         gather and validate inputs
├── outputs.ts        persistence
├── prompts/          versioned prompt files + schemas + meta + evals
└── __tests__/
```

**Rules:** engines never import other engines. Engines never contain scoring maths — they call `packages/domain`. Engines are the only place adapters and the database meet.

---

## 6. `packages/adapters`

```
packages/adapters/
├── ai/           tier→model map, structured generate, embed, vision, cost table
├── ideogram/     generation, upscale, style templates
├── etsy/         OAuth, listings, images, inventory, publish, stats
├── printify/     catalogue, uploads, products, mockups, publish bridge
├── marketdata/   MarketDataProvider chain: csv · session · api · etsy-public · fixture
├── trademark/    USPTO · EUIPO · UKIPO clients with caching
├── storage/      S3-compatible object storage, signed URLs
├── imaging/      sharp pipelines, background removal, upscale, vectorise
└── email/        transactional notifications
```

Every adapter has the same internal shape (`index.ts`, `client.ts`, `mappers.ts`, `errors.ts`, `schemas.ts`, `fixtures/`, `__tests__/`) and obeys the universal adapter contract in [doc 9 §6](09-service-architecture.md).

**Enforced by lint:** no vendor SDK may be imported outside its own adapter package.

---

## 7. `packages/db`

```
packages/db/
├── prisma/
│   ├── schema.prisma
│   ├── migrations/
│   └── seed.ts                 realistic fixture workspace (doc 9 §10)
├── src/
│   ├── client.ts               read/write split clients, RLS session variables
│   ├── repositories/           one per aggregate; the ONLY place Prisma is called
│   ├── transactions.ts
│   ├── partitions.ts           partition creation/rotation helpers
│   └── vector.ts               pgvector query helpers
```

**Rule:** application code never imports Prisma directly. It calls a repository. This is what makes the read/write split, tenancy enforcement and query instrumentation universal rather than per-call-site.

---

## 8. Dependency rules (machine-enforced)

`tooling/dependency-cruiser.js`:

```
apps/*            → packages/*                                  ✅
packages/engines  → domain, db, adapters, contracts, queue,
                    observability                               ✅
packages/domain   → contracts                                   ✅ (only)
packages/domain   → db, adapters, queue, any I/O                ❌
packages/adapters → contracts, observability                    ✅
packages/adapters → engines, db                                 ❌
packages/engines  → packages/engines                            ❌
any               → app-internal paths of another package       ❌
vendor SDK        → outside its own adapter                     ❌
```

Violations fail CI. The rule file is the architecture's enforcement mechanism; without it, the boundaries erode within a quarter.

---

## 9. Naming conventions

| Thing | Convention | Example |
|---|---|---|
| Files | kebab-case | `success-factors.ts` |
| React components | PascalCase file and export | `ScoreDial.tsx` |
| Hooks | `use` prefix | `useRunProgress.ts` |
| Types/interfaces | PascalCase, no `I` prefix | `SuccessFactor` |
| Zod schemas | PascalCase + `Schema` | `SuccessFactorSchema` |
| Enums | PascalCase type, snake_case values (matching Postgres) | `PaletteFamily.muted_green` |
| DB tables/columns | snake_case, plural tables | `success_factors.cohort_support_pct` |
| Queue jobs | dot-namespaced | `artwork.generate` |
| tRPC procedures | camelCase, verb-first | `regenerateOne` |
| Prompt ids | kebab-case + semver | `concept-generation-success/v1.2.0` |
| Feature flags | snake_case with domain prefix | `engine_gap_v2` |
| Env vars | SCREAMING_SNAKE | `RUN_CONCURRENCY_LIMIT` |
| Money fields | `*_amount` + `*_currency` | `retail_price_amount` |
| Booleans | `is_`/`has_`/`can_` | `is_bestseller` |
| Timestamps | `*_at` | `published_at` |

---

## 10. Testing layout

```
packages/domain/src/**/__tests__/          unit + property (Vitest + fast-check)
packages/adapters/*/__tests__/             fixture-backed integration
packages/engines/*/__tests__/              engine tests with mocked adapters
packages/engines/*/prompts/*/evals/        AI eval suites
apps/web/tests/                            component + route tests
e2e/                                       Playwright: five critical journeys
infra/scripts/load/                        k6 scenarios
```

| Layer | Target coverage | Runs |
|---|---|---|
| `packages/domain` | ≥ 80% lines, 100% of scoring functions | Every PR |
| Adapters | Every endpoint + every documented error | Every PR |
| Engines | Happy path + each degradation path | Every PR |
| AI evals | Fast suite per PR, full suite nightly | PR / nightly |
| E2E | 5 journeys | Pre-merge |
| Load | Nightly | Nightly |

---

## 11. Turborepo pipeline

```jsonc
{
  "tasks": {
    "build":     { "dependsOn": ["^build"], "outputs": ["dist/**", ".next/**"] },
    "typecheck": { "dependsOn": ["^build"] },
    "lint":      {},
    "test":      { "dependsOn": ["^build"], "outputs": ["coverage/**"] },
    "test:eval": { "dependsOn": ["^build"], "cache": false },
    "e2e":       { "dependsOn": ["build"], "cache": false },
    "db:migrate":{ "cache": false },
    "dev":       { "cache": false, "persistent": true }
  }
}
```

Remote caching enabled in CI so unchanged packages are never rebuilt — the single largest CI time saving in a monorepo of this shape.

---

## 12. Where a new feature goes

A worked example — adding a "Bundle Recommendation Engine" that suggests multi-product bundles:

1. **Types** → `packages/contracts/src/bundle.ts`
2. **Maths** → `packages/domain/src/scoring/bundle.ts` + tests
3. **Engine** → `packages/engines/bundle/` with `execute.ts`, prompts, tests
4. **Persistence** → `packages/db` migration + `repositories/bundle.ts`
5. **DAG** → add a `StepDefinition` in `packages/orchestrator/dag.ts`
6. **Queue** → register the processor in `apps/worker/registry.ts`
7. **API** → `apps/web/src/server/trpc/routers/bundle.ts`
8. **UI** → route + components under `apps/web/src/app/(app)/...`
9. **Flag** → `feature_flags` row, default off
10. **Docs** → update FR list, ERD if schema changed, ADR if a boundary moved

**Ten mechanical steps, each in an obvious place.** That predictability is the point of the structure.
