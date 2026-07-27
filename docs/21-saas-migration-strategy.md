# 21 — Future SaaS Migration Strategy

**Version:** 1.0

---

## 1. The thesis

The single-user build is not a prototype that gets thrown away. It is **the product, running with one tenant**. Because tenancy, credential isolation, rate limiting, budgets and audit were built at the start, the SaaS conversion is an *additive* four-week project rather than a rewrite.

**What is already done (Phase 1–5):**

| Requirement | Status |
|---|---|
| `workspace_id` on every table | ✅ day one |
| Composite indexes leading with `workspace_id` | ✅ day one |
| `workspace_members` with a role enum | ✅ day one, one row |
| RLS policies written and CI-tested | ✅ day one, enforcement flipped at Phase 6 |
| Per-workspace credential encryption with per-workspace DEKs | ✅ day one |
| Per-workspace rate limiting keys | ✅ day one |
| Per-workspace budgets and cost ledgers | ✅ day one |
| Audit log with actor attribution | ✅ day one |
| Stateless web and worker tiers | ✅ day one |
| Queue abstraction supporting per-tenant partitioning | ✅ day one |
| Cursor pagination everywhere | ✅ day one |
| Feature flags with workspace targeting | ✅ day one |

**What Phase 6 adds:** invitations and roles enforcement, Stripe billing and metering, plan entitlements, per-tenant fair scheduling, shared market-fact caching, the public API, and the commercial surface (onboarding, docs, legal).

---

## 2. Migration steps

### Step 1 — Enable tenant isolation enforcement (2 days)

```sql
-- Policies already exist. Switch the application role.
CREATE ROLE app_tenant NOLOGIN;
GRANT app_tenant TO app_user;
ALTER ROLE app_user NOBYPASSRLS;
```

The connection pool already sets `app.workspace_id` on checkout. The change is a role grant plus a deployment.

**Verification (blocking):** an isolation test suite that, for every tRPC procedure taking an entity id, attempts access with an id belonging to a different workspace and asserts `NOT_FOUND` — no exceptions, no timing differences. This suite runs on every PR thereafter.

### Step 2 — Multi-user (4 days)

- Invitation flow: email invite, tokenised acceptance, role assignment.
- Role enforcement middleware using the permission matrix from [doc 15 §4](15-security-architecture.md) — the matrix exists; only the enforcement branch is new.
- Member management UI; last-owner protection; seat counting.
- Per-user activity attribution already exists via `created_by_user_id` and the audit log.

### Step 3 — Billing (5 days)

```sql
CREATE TABLE plans (
  id uuid PRIMARY KEY, code text UNIQUE, name text,
  stripe_price_id text, interval text,
  price_amount bigint, currency char(3),
  entitlements jsonb NOT NULL      -- limits and feature gates
);

CREATE TABLE subscriptions (
  id uuid PRIMARY KEY, workspace_id uuid UNIQUE REFERENCES workspaces(id),
  plan_id uuid REFERENCES plans(id),
  stripe_customer_id text, stripe_subscription_id text,
  status text,                     -- trialing|active|past_due|canceled|paused
  current_period_start timestamptz, current_period_end timestamptz,
  cancel_at_period_end boolean DEFAULT false,
  trial_ends_at timestamptz,
  created_at timestamptz DEFAULT now(), updated_at timestamptz DEFAULT now()
);

CREATE TABLE usage_events (      -- partitioned monthly
  id uuid NOT NULL, workspace_id uuid NOT NULL,
  metric text NOT NULL,          -- run|artwork|published_product|ai_spend|seat
  quantity numeric(14,4) NOT NULL,
  cost_amount bigint,            -- our marginal cost, for margin analysis
  entity_type text, entity_id uuid,
  billing_period date NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

CREATE TABLE entitlement_usage (   -- fast counters for enforcement
  workspace_id uuid, metric text, billing_period date,
  used numeric(14,4) NOT NULL DEFAULT 0, limit_value numeric(14,4),
  PRIMARY KEY (workspace_id, metric, billing_period)
);
```

- Stripe Checkout for signup, Customer Portal for self-service management.
- Webhooks: `checkout.session.completed`, `customer.subscription.updated/deleted`, `invoice.payment_failed/succeeded`.
- Entitlement checks in the existing `spendProcedure` middleware — the hook already exists for budgets; plan limits slot in beside it.
- Dunning: 3 retries, then downgrade to read-only (never delete data).

### Step 4 — Per-tenant provider credentials (2 days)

Already structurally supported: `integrations` is per-workspace. The change is UX (each tenant connects their own Etsy and Printify) and, for AI/Ideogram, a choice between platform keys with metering (default) and bring-your-own-key on higher plans.

**Consequence:** Etsy and Printify quota scales with tenant count rather than being a shared ceiling — the single most important scaling property of this model.

### Step 5 — Fair scheduling and shared caching (4 days)

- Per-tenant sub-queues with weighted round-robin and plan-derived concurrency caps.
- Shared market-fact cache: `style_profiles` keyed by `(image_hash, extractor_version)` and trademark results by normalised term, promoted to a platform-level table readable by all tenants. **Strict boundary:** only content-addressed market facts are shared. Scores, cohorts, factors, gaps, concepts and anything derived from a tenant's own configuration remain tenant-scoped. This is enforced by table separation, not by a query filter.

### Step 6 — Public API (5 days)

REST `/v1` façade over the existing services (doc 8 §8), API keys with scopes, OpenAPI 3.1 generated in CI, outbound webhooks with HMAC signing and retry.

### Step 7 — Commercial surface (5 days)

Onboarding, marketing site, documentation, support tooling, terms, privacy policy, subprocessor page, DPAs, and a penetration test.

---

## 3. Pricing model

Derived from the unit economics in [doc 16 §8](16-scalability-architecture.md), not from competitor guessing. Marginal cost per published product is £0.90–£3.40 depending on depth and retries; that sets the floor.

| Plan | Price/mo | Runs | Artworks | Published | Seats | Target |
|---|---|---|---|---|---|---|
| **Research** | £29 | 10 standard | 0 | — | 1 | P4 Newcomer — the freemium-adjacent hook; research only, no creation |
| **Solo** | £79 | 30 | 100 | 60 | 1 | P1 Solo Operator — the core plan |
| **Studio** | £199 | 100 | 400 | 250 | 3 | P2 Scaling Seller |
| **Agency** | £499 | 400 | 1,500 | 1,000 | 10 | P3 Agency, multi-client workspaces |
| **Enterprise** | custom | custom | custom | custom | custom | BYO keys, SSO, SLA, dedicated support |

**Overages:** £0.90/run, £0.12/artwork beyond plan. Metered, shown live, capped by a tenant-set spend limit that defaults to the plan price (so nobody receives a surprise bill).

**Gross margin at these prices:** Solo at full utilisation ≈ £79 revenue against ≈ £34 marginal cost = **57%**. With shared market-fact caching at scale, ≈ **68%**. The Research plan is deliberately near-breakeven — it is an acquisition instrument, and it is the plan with the lowest marginal cost because it never generates images.

**Deliberate choices:**
- **No free tier.** Every run costs real money in AI and image credits; a free tier would be a subsidy for abuse. A 7-day trial with a hard cap of 3 runs replaces it.
- **Usage components are essential.** A flat-rate plan with unlimited runs is how AI products go bankrupt.
- **Annual billing at 2 months free** to improve cash conversion.

---

## 4. Team accounts

| Capability | Implementation |
|---|---|
| Roles | The existing `workspace_role` enum with the permission matrix from doc 15 §4 |
| Multiple workspaces per user | `workspace_members` already supports it; add a workspace switcher and per-workspace session scoping |
| Approval workflows | A `member` produces drafts; an `admin`/`owner` publishes. This maps exactly onto the existing Final Review gate — no new concept required. |
| Shared research | Reports are already workspace-scoped; visibility is automatic |
| Attribution | `created_by_user_id` and the audit log already record who did what |
| Client workspaces (agencies) | One workspace per client; the user belongs to many; billing rolls up to the paying account |

**Notable:** the approval workflow that agencies will ask for already exists as the human publish gate. It was built for safety and turns out to be the exact primitive teams need. That is what designing the boundaries well buys you.

---

## 5. What must change in the product, not just the platform

Multi-tenancy is the easy part. These are the harder product changes:

| Change | Why |
|---|---|
| **Onboarding without a technical operator** | Phase 1 assumes someone who can generate a Printify token. SaaS users cannot. Needs guided, forgiving connection flows with excellent error recovery. |
| **Data-source guidance** | The market data provider chain is subtle. Users need a clear, opinionated recommendation and a one-click CSV import, not a menu of adapters. |
| **Defaults over configuration** | The fee model, margin floor, scoring weights and budgets are all operator-tuned in Phase 1. SaaS needs sane, regional defaults and progressive disclosure of the knobs. |
| **Cross-tenant benchmarking** | "Your opportunity score of 68 is in the top 30% of gardening niches researched on this platform" is a feature only a multi-tenant product can offer, and it is a genuine differentiator. Requires careful aggregation and an explicit privacy boundary. |
| **Support surface** | Correlation ids, run inspection tooling, and an admin view to diagnose a tenant's failed run without accessing their data content. |
| **Abuse prevention** | Spend caps, anomaly detection, and a policy on what the legal engine will refuse regardless of user override. |
| **Onboarding-to-value time** | Currently a run takes 9 minutes. A trial user needs a compelling partial result in under 60 seconds — likely a pre-computed sample report for popular niches. |

---

## 6. Data migration for existing tenants

The Phase 1 operator becomes tenant #1 with zero migration: their workspace row already exists, their data is already scoped, and their credentials are already per-workspace encrypted. The only change is that `workspace_members` gains meaningful roles and a `subscriptions` row is created with a grandfathered plan.

**This is the payoff for the day-one tenancy decision.** The alternative — a `workspace_id` backfill across 25 tables with terabytes of snapshot data, plus rewriting every query — is a multi-month project with real risk of data corruption.

---

## 7. Go-to-market sequence

| Stage | Audience | Motion |
|---|---|---|
| **Design partners (5–10)** | Sellers the operator knows personally | Free access in exchange for weekly feedback and outcome data. The outcome data is worth more than the revenue at this stage. |
| **Private beta (50)** | POD communities, Discord, subreddits | Waitlist, invite-only, £29 Research plan as the entry point |
| **Public launch** | Etsy seller community | Content marketing built on the product's own output: published niche opportunity reports as lead magnets — the product markets itself by doing its job in public |
| **Growth** | Scaling sellers, agencies | Case studies with real revenue attribution, affiliate programme, integration partnerships |

**The strongest acquisition asset** is the system's own analysis. Publishing "The 20 most underserved Etsy sub-niches this quarter, with evidence" costs one run and demonstrates the entire value proposition.

---

## 8. Metrics for the SaaS business

| Metric | Target at 12 months |
|---|---|
| Paying workspaces | 300 |
| MRR | £26,000 |
| Gross margin | ≥ 60% |
| Net revenue retention | ≥ 105% |
| Monthly logo churn | ≤ 5% |
| Trial → paid conversion | ≥ 22% |
| Time to first published product | ≤ 48 h from signup |
| CAC payback | ≤ 5 months |
| Cost per active workspace | ≤ £28/mo |
| Activation (first run completed) | ≥ 70% of signups |

**The metric that actually predicts retention:** products published per workspace per month. A user who publishes is a user who renews. Everything in the onboarding funnel should optimise for the first published product, not for the first login.

---

## 9. Sequencing decision

**Do not build Phase 6 until the Phase 6 gate is met** (doc 19 §7): one month of real use, 30 products published, and a one-sentence value articulation that a stranger would pay for.

The temptation to build billing early is strong because it feels like progress toward a business. It is not. The business is the loop between evidence and outcome, and that loop must be proven with one user before it is sold to a hundred.

**Recommended sequence:**
1. Weeks 1–19: build Phases 1–5.
2. Weeks 20–23: use it. Publish 30+ products. Measure whether they outperform the manual baseline.
3. Week 24: evaluate against the gate honestly.
4. If met: Phases 6 and go-to-market. If not: fix the product, or stop.
