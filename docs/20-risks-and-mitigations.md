# 20 — Risks & Mitigations

**Version:** 1.0

Scoring: **Likelihood** and **Impact** each 1–5; **Score** = L × I. Anything ≥ 15 requires an explicit, funded mitigation before the phase that exposes it.

---

## 1. Risk register — summary

| ID | Risk | L | I | Score | Category | Phase exposed |
|---|---|---|---|---|---|---|
| R-01 | Market data access (EverBee) unavailable or legally constrained | 4 | 4 | **16** | External | 2 |
| R-02 | AI cost per useful design exceeds product margin | 3 | 5 | **15** | Economic | 2–3 |
| R-03 | Etsy policy change on AI-generated products or API listing creation | 3 | 5 | **15** | Regulatory | 4 |
| R-04 | Trademark/copyright infringement despite screening | 2 | 5 | 10 | Legal | 3 |
| R-05 | Duplicate listings or duplicate charges from retry logic | 2 | 5 | 10 | Technical | 4 |
| R-06 | Statistical conclusions are spurious (small n, survivorship, confounding) | 4 | 4 | **16** | Product | 2 |
| R-07 | Learning loop never reaches useful sample size | 4 | 3 | 12 | Product | 5 |
| R-08 | Generated artwork quality below commercial standard | 3 | 4 | 12 | Product | 3 |
| R-09 | Provider API changes break integrations | 4 | 3 | 12 | External | 2–4 |
| R-10 | Scope creep into a general AI store builder | 4 | 4 | **16** | Delivery | all |
| R-11 | Single-developer bus factor | 3 | 4 | 12 | Delivery | all |
| R-12 | Credential compromise (Etsy write access) | 2 | 5 | 10 | Security | all |
| R-13 | Prompt injection via competitor listing text | 3 | 3 | 9 | Security | 2 |
| R-14 | Estimated sales figures are materially wrong, misleading decisions | 4 | 4 | **16** | Product | 2 |
| R-15 | Etsy rate limits constrain research or publishing | 3 | 3 | 9 | Technical | 4 |
| R-16 | Printify cost changes silently erode margin | 3 | 4 | 12 | Economic | 4 |
| R-17 | Ideogram unavailable, repriced, or terms-changed | 3 | 4 | 12 | External | 3 |
| R-18 | Database growth outpaces cost expectations | 2 | 3 | 6 | Technical | 5+ |
| R-19 | Operator over-trusts the system and stops applying judgement | 3 | 4 | 12 | Product | 4+ |
| R-20 | SaaS build begins before product-market fit is demonstrated | 4 | 4 | **16** | Strategic | 6 |
| R-21 | Competitor (EverBee/Alura) ships the same feature set | 3 | 4 | 12 | Market | post-launch |
| R-22 | Etsy marketplace itself declines or changes economics | 2 | 5 | 10 | Market | post-launch |
| R-23 | Multi-tenant data leakage after SaaS conversion | 2 | 5 | 10 | Security | 6 |
| R-24 | AI provider outage halts the pipeline | 3 | 3 | 9 | External | 2+ |
| R-25 | Design homogenisation — the system converges on one look | 3 | 3 | 9 | Product | 3+ |

---

## 2. Critical risks (score ≥ 15) — detailed treatment

### R-01 · Market data access unavailable or legally constrained (16)

**Description.** EverBee publishes no general public API. Automated access to its web application may conflict with its terms. If the highest-value data input disappears, sales and revenue estimates disappear with it.

**Mitigations (all committed, not optional):**
1. **Provider abstraction with a guaranteed path.** `MarketDataProvider` chain with `everbee_csv` (operator's own export) as the default. This path cannot be broken by any external party. (doc 11)
2. **Capability-driven degradation.** Every engine reads capabilities and adapts. A run with no estimates still produces style, pricing, SEO and gap analysis — the genuinely differentiated 70% of the value.
3. **Proxy models.** Review-count-derived sales proxies, calibrated in Phase 5 against the operator's own realised sales, materially closing the fidelity gap over time.
4. **Compliance posture.** Amber adapters are off by default, require consent, are rate-limited, and never implement evasion. If access requires evasion, the adapter is unavailable — a stated product constraint.
5. **Commercial route.** Approach EverBee for API/partner access in Phase 2 and record the outcome.

**Residual risk:** medium. Accepted, because the CSV path is genuinely sufficient.
**Trigger to escalate:** operator's EverBee plan removes export capability.

### R-02 · AI cost exceeds margin (15)

**Description.** At £1.10 per research run and £0.42 per accepted design, a heavy month is manageable. At 3× those figures, the SaaS unit economics invert.

**Mitigations:**
1. Hard per-run and per-workspace budgets enforced *before* spend (FR-010, doc 9 §7).
2. Aggregate-before-prompting: ~85% token reduction versus naive approaches (doc 10 §7.1).
3. Model tiering with promotion only on eval evidence.
4. Content-hash caching for vision (40–70% hit rate on repeat niches) and registry lookups.
5. Prompt caching across multi-call steps.
6. Concept-before-artwork gating: ~75% of image spend eliminated.
7. Cost regression tests in CI — a prompt change that raises mean tokens by >15% fails the build.
8. Shared market-fact caching at SaaS scale (roughly doubles gross margin — doc 16 §8).

**Residual risk:** low-medium. Monitored continuously via cost telemetry.
**Trigger to escalate:** cost per run drifting >15% above budget for two consecutive weeks.

### R-03 · Etsy policy change (15)

**Description.** Etsy could restrict AI-generated products, require disclosure, change API listing-creation rules, or alter its fee structure.

**Mitigations:**
1. **Draft-only automation with a mandatory human gate.** The system never publishes autonomously, which keeps the operator — a human seller — in the compliance loop.
2. **Full provenance retained** for every design (brief, prompt, model, human edits), so any disclosure requirement can be met retrospectively.
3. **Configurable AI disclosure** in listing descriptions.
4. **Correct policy metadata** on every listing: `who_made`, `when_made`, `is_supply`, production partner declared.
5. **Marketplace abstraction** designed (not built) so Amazon Merch / Shopify / Redbubble can be added without touching the intelligence pipeline.
6. **Fee model is configuration**, so a fee change is a settings update, not a code change.

**Residual risk:** medium. Inherent to building on someone else's marketplace.
**Trigger to escalate:** any Etsy policy announcement touching AI content or API listing creation.

### R-06 · Spurious statistical conclusions (16)

**Description.** This is the most insidious risk in the product. With 400 listings and 30 attributes, some attribute will always correlate with success by chance. A confident, well-designed report showing a false pattern is *worse than no report* — it actively misleads.

**Mitigations:**
1. **Minimum sample sizes.** Factors with n < 8 in the cohort are suppressed from ranking and flagged `insufficient_evidence` (FR-405).
2. **Significance testing.** Two-proportion z-tests, χ², Cliff's delta; p > 0.10 suppressed.
3. **Baseline always shown.** Every factor displays cohort support *and* baseline support *and* lift. A bare "84% use green" is not permitted by the UI contract (FR-411).
4. **Confidence chips** on every factor, derived from n and p, not from vibes.
5. **Multicollinearity flagging** in the correlation matrix.
6. **Causality labelling** in the failure report distinguishes causal-plausible from correlational-only (FR-505).
7. **Survivorship honesty.** Cohorts are age-normalised; listings with insufficient exposure are excluded and the exclusion is reported.
8. **Prediction calibration** in Phase 5 measures which factors actually predicted outcomes — the ultimate audit of the statistics (FR-1616).
9. **Multiple-comparison awareness:** the number of attributes tested is reported alongside the factor list, so the operator can weigh the family-wise error risk.

**Residual risk:** medium. Partially irreducible with observational data — mitigated by honesty rather than eliminated.

### R-10 · Scope creep (16)

**Description.** The adjacent-feature pull is enormous: Amazon Merch, Shopify, ad management, order fulfilment, customer service, a design marketplace, a template library. Each is individually reasonable and collectively fatal.

**Mitigations:**
1. Explicit in/out scope in doc 1 §5, with an "architected-for but not built" list that gives ideas somewhere to go without entering the build.
2. Phase gates with exit criteria; no phase starts until the previous one meets them.
3. Every new capability requires an ADR stating what it displaces.
4. The roadmap's standalone-value property means stopping early is always an option, reducing the pressure to bundle.
5. A single product principle governs: *does this improve the decision-to-outcome loop for Etsy POD?* If not, it is a different product.

### R-14 · Estimated sales figures materially wrong (16)

**Description.** Third-party sales estimates carry substantial error, and every downstream cohort, factor and score inherits it.

**Mitigations:**
1. **Never present estimates as facts.** `EstimateBadge` with source and confidence on every estimated value (FR-006), enforced as a UI contract.
2. **Confidence propagation.** A low-confidence input produces a low-confidence output, visibly.
3. **Rank-based rather than value-based analysis.** Cohorts use percentiles, and lift uses proportions — both robust to systematic estimation bias, which mostly cancels when it affects all listings similarly.
4. **Multi-source merge** with field-level provenance, so a measured value always beats an estimate.
5. **Phase 5 calibration** against the operator's own realised sales converts a guess into a fitted proxy.
6. **The differentiated analysis does not depend on estimates.** Style, layout, pricing, image-count and SEO analysis all work on measured Etsy data.

### R-20 · SaaS build before product-market fit (16)

**Description.** Building billing, teams, metering and a public API before the single-user product has demonstrably worked is the classic way to spend four weeks on infrastructure for a product nobody wants.

**Mitigations:**
1. **Hard gate on Phase 6:** ≥ 1 month of real use, ≥ 30 products published through the system, and a one-sentence articulation of the value a stranger would pay for.
2. Multi-tenancy is *architectural* from day one (one column) but *functional* only at Phase 6 — the cheap part is done early, the expensive part is deferred.
3. Phase 1–5 deliver a complete, valuable product independent of any SaaS ambition.

---

## 3. Remaining risks — mitigation summary

| ID | Mitigation |
|---|---|
| **R-04 Infringement** | Mandatory pre-generation legal gate; three registries; deterministic risk rule table; adversarial eval with a ≥ 0.98 recall release gate; artwork re-screen; listing-text screen; 7-year screening records; burned-terms list; step-up auth on overrides; explicit "not legal advice" disclaimer |
| **R-05 Duplicates** | Four-layer defence: application idempotency keys, pre-flight existence checks, reconciliation on ambiguous timeout, database uniqueness. Chaos test asserts zero duplicates under worker kill (doc 13 §8, doc 14 §7) |
| **R-07 Learning loop starved** | Expert priors are useful on day one; the loop refuses to fit below n=50 and says so; shrinkage toward the prior prevents small-sample overreaction; proposals are never auto-activated |
| **R-08 Artwork quality** | Detailed briefs from real style statistics; 4 variants; print-readiness QA with specific remedies; regeneration with steer; operator acceptance gate; provider portability if quality is structurally insufficient |
| **R-09 Provider API changes** | Adapter isolation; schema validation that fails loudly rather than mis-parsing; fixture-based contract tests; circuit breakers; documented fallbacks per provider |
| **R-11 Bus factor** | This specification set; ADRs for every non-obvious decision; ≥ 80% coverage on domain logic; runbooks for every alert; one-command local environment; conventional commits and readable history |
| **R-12 Credential compromise** | Envelope encryption with per-workspace DEKs; least-privilege scopes; mandatory TOTP; kill switches tested quarterly; audit logging; incident runbook with a 15-minute containment target |
| **R-13 Prompt injection** | Structural quarantine of untrusted text; no tool access on untrusted-input prompts; enum-validated output; **the generative stage never sees raw external text** — injection has no path to influence creation; injection-resistance eval suite |
| **R-15 Rate limits** | Token buckets with 15% reserved publish quota; priority classes; conditional requests; caching; per-tenant credentials at SaaS scale |
| **R-16 Printify cost drift** | Daily variant cost sync; >5% change alert; automatic repricing of unpublished drafts; portfolio-wide margin recheck; Phase 5 order reconciliation against actual charges |
| **R-17 Ideogram risk** | `ImageGenerationProvider` interface; ~80% of the artwork pipeline is provider-agnostic; swap is an adapter plus style-template mappings plus an eval run |
| **R-18 Database growth** | Partitioning from day one; 180-day raw retention with Parquet tiering; content-hash dedupe; aggregates retained, raw discarded; monthly growth alerting |
| **R-19 Over-trust** | Every score shows its evidence and confidence; degraded results are visibly degraded; human gates at concept, legal and publish; calibration dashboard shows where predictions were wrong; the UI never says "will sell", only "predicted" |
| **R-21 Competitive copying** | The moat is the proprietary outcome dataset, not the features. A competitor shipping concept generation starts at zero outcomes while we start with N shops × M months. Speed to accumulate data matters more than feature parity |
| **R-22 Etsy decline** | Marketplace abstraction designed; the intelligence pipeline is marketplace-agnostic in principle; the analysis capability transfers to any listing-based marketplace |
| **R-23 Tenant leakage** | Workspace scoping at the repository layer *and* Postgres RLS; a cross-tenant isolation test suite that attempts access by id from another workspace on every endpoint; `NOT_FOUND` responses that never confirm existence |
| **R-24 AI outage** | Circuit breakers; steps move to `blocked_external` and resume automatically; runs are never lost; tier remapping via config allows switching model families quickly |
| **R-25 Design homogenisation** | Within-run embedding dedupe (>0.92 rejected); cross-history dedupe (>0.95 flagged); palette perturbation within family bounds; gap-derived concepts explicitly pull away from the modal style; originality scoring against the workspace's own prior artwork |

---

## 4. Risk review cadence

| Activity | Frequency | Output |
|---|---|---|
| Register review | Every phase gate | Re-scored risks, new entries, closed entries |
| Cost telemetry review | Weekly | Drift against budgets; action if > 15% |
| Provider health review | Weekly | Breaker events, error rates, quota trends |
| Terms & policy review (Etsy, Printify, market data, AI) | Every phase gate | Compliance date recorded in doc 11 |
| Security review | Every phase gate | Findings triaged to closure |
| Statistical validity audit | After each 50 outcomes | Which factors predicted; which did not; adjust thresholds |
| Restore drill | Monthly | Recorded result |
| Kill-switch test | Quarterly | Recorded result |

---

## 5. Things that would justify stopping

Stated deliberately, because a plan without stopping conditions is a plan to keep spending.

1. **After Phase 2:** if the reports do not change the operator's niche decisions — if they confirm what was already known — the core thesis is wrong and the remaining phases are not worth building.
2. **After Phase 3:** if generated artwork requires more manual rework than designing from scratch, the creative half of the product is not viable and the system should be repositioned as research-only.
3. **After Phase 4:** if published products from the system do not outperform the operator's manually-created baseline within 90 days, the loop does not close and the learning phase will not rescue it.
4. **Before Phase 6:** if the operator cannot name three specific people who would pay for this, there is no SaaS to build.

Each is a legitimate outcome, and each saves more than it costs to discover.
