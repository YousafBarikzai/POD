# 01 — Product Requirements Document

**Version:** 1.0 · **Owner:** Product/CTO · **Status:** Approved

---

## 1. Product vision

> **POD Intelligence turns Etsy print-on-demand from a creative gamble into a repeatable, evidence-driven manufacturing process for attention.**

A single operator should be able to sit down, type "Fishing Hoodies", and within fifteen minutes have: a defensible answer on whether that market is worth entering, a quantified picture of who is winning and why, twenty original concept directions grounded in that evidence, legally screened artwork, and a fully SEO'd Etsy draft listing awaiting one click of approval.

## 2. Positioning

| | EverBee / eRank / Alura | Midjourney / Ideogram | Printify / Printful | Placeit / Kittl | **POD Intelligence** |
|---|---|---|---|---|---|
| Market research | ✅ raw data | ❌ | ❌ | ❌ | ✅ **synthesised into decisions** |
| Visual style analytics | ❌ | ❌ | ❌ | ❌ | ✅ |
| Failure analysis | ❌ | ❌ | ❌ | ❌ | ✅ |
| Gap detection | partial | ❌ | ❌ | ❌ | ✅ |
| Concept generation grounded in data | ❌ | ❌ | ❌ | ❌ | ✅ |
| Artwork generation | ❌ | ✅ | ❌ | ✅ templates | ✅ **brief-driven, POD-validated** |
| Legal screening | ❌ | ❌ | ❌ | ❌ | ✅ |
| SEO generation | partial | ❌ | ❌ | ❌ | ✅ |
| Publishing | ❌ | ❌ | ✅ | ❌ | ✅ |
| Outcome learning | ❌ | ❌ | ❌ | ❌ | ✅ |

**The wedge:** every competitor owns one link in the chain. POD Intelligence owns the *chain*, and the chain is where the compounding data asset lives.

**Category definition:** "POD Market Intelligence & Production System". Not a research tool, not a design tool — a decision system.

## 3. Personas

### P1 — "Sam", the Solo POD Operator *(primary, Phase 1)*
- Runs one Etsy shop, 150–600 listings, £1k–£8k/month revenue.
- Spends 60% of their working time on research and design, 40% on operations.
- Technically comfortable (uses EverBee, Canva, Printify) but not a developer.
- **Pain:** "I have no idea which of my next ten designs will be the one that works, so I make all ten and hope."
- **Success looks like:** publishing 30% fewer products but earning 2× more, because each product was chosen with evidence.
- **Uses:** the entire application, daily.

### P2 — "Jordan", the Scaling Seller *(SaaS Stage 2)*
- 3–8 shops or one large shop, £15k–£60k/month, 1–2 VAs.
- Needs delegation: research done once, executed by assistants.
- **Pain:** cannot transfer their intuition to their team.
- **Needs from us:** shareable reports, saved briefs, role separation, bulk operations, approval queues.

### P3 — "Alex", the Agency / Brand Studio *(SaaS Stage 3)*
- Manages POD for multiple clients.
- **Needs from us:** workspaces per client, white-labelled reports, seat-based billing, audit trails, API access.

### P4 — "Riley", the Data-Curious Newcomer *(SaaS Stage 2, acquisition)*
- No shop yet, evaluating niches.
- **Needs from us:** the Opportunity Report as a standalone, low-cost entry product. This is the freemium hook.

Phase 1 builds exclusively for **P1**. Every P2–P4 need is accommodated *architecturally* (see doc 21) but not built.

## 4. Jobs To Be Done

| ID | Job statement | Served by |
|---|---|---|
| JTBD-1 | *When I'm considering a niche, I want to know whether it's worth my time before I invest, so I don't waste weeks on a saturated market.* | Opportunity Report Engine (Step 2) |
| JTBD-2 | *When I enter a market, I want to know exactly who is winning and what they're doing, so I can compete on evidence.* | Competitor Analysis Engine (Step 3) |
| JTBD-3 | *When I design, I want to know which visual choices correlate with sales in this specific niche, so my aesthetics are strategic.* | Success Analysis Engine (Step 4) |
| JTBD-4 | *When I design, I want to know what reliably fails, so I stop repeating other people's mistakes.* | Failure Analysis Engine (Step 5) |
| JTBD-5 | *When everyone is fighting over the same sub-niche, I want to find the one nobody has covered, so I can own it.* | Market Gap Engine (Step 6) |
| JTBD-6 | *When I have an idea, I want a prediction of how it will perform before I spend money making it.* | Design Success Predictor (Step 7) |
| JTBD-7 | *When I need ideas, I want original concepts derived from my market data — not generic AI slop.* | Concept Generator (Step 8) |
| JTBD-8 | *When I create, I want to be certain I'm not infringing anyone's rights, so I never lose my shop.* | Legal & Safety Engine (Step 9) |
| JTBD-9 | *When a concept is approved, I want production-ready artwork without a designer.* | Artwork Pipeline (Step 10) |
| JTBD-10 | *When I choose a product, I want the blueprint/variant combination with the best margin-to-demand ratio.* | Product Recommendation Engine (Step 11) |
| JTBD-11 | *When I write a listing, I want SEO that reflects how buyers actually search in this niche.* | SEO Engine (Step 12) |
| JTBD-12 | *When I publish, I want it to take one minute, not thirty, and I want to check it before it goes live.* | Printify + Etsy integration, Final Review (Steps 13–15) |
| JTBD-13 | *When products sell (or don't), I want the system to get smarter, so my advantage compounds.* | Analytics + Learning Loop (Step 16) |

## 5. Scope — Phase 1 (single user)

### 5.1 In scope

| Area | Included |
|---|---|
| Users | Exactly one operator account, password + TOTP |
| Shops | One Etsy shop connection, one Printify account |
| Markets | Etsy only |
| Locales | English (en-GB / en-US) listings |
| Currency | GBP and USD, stored as minor units |
| Product types | T-shirt, sweatshirt, hoodie, mug, poster/art print, tote bag |
| Research | Full 16-step pipeline |
| Concurrency | 3 simultaneous research runs |
| Data volume | 200 shops, 40k listings, 5k concepts, 2k artworks, 1k published listings |
| Retention | Indefinite for owned entities; 180 days for raw competitor snapshots |

### 5.2 Out of scope (Phase 1)

Multi-user · billing · teams · mobile native app · other marketplaces · ads management · order fulfilment operations · customer messaging · non-English listings · video/animated assets · physical sample ordering · bulk CSV listing upload to Etsy · Etsy Ads API.

### 5.3 Architected-for, deferred

Multi-tenancy · Stripe subscriptions & metering · role-based access · public REST API + API keys · marketplace-agnostic listing abstraction · white-label reports · webhook fan-out to customer systems · model fine-tuning on proprietary outcome data.

## 6. Product principles

1. **Evidence over opinion.** Every recommendation displays the data that produced it. No unexplained numbers. A score without a "why" is a bug.
2. **Cheap before expensive.** Concepts (text, ~£0.02) are always generated and gated before artwork (image, ~£0.06). Never spend image credits on an unvalidated idea.
3. **Human owns the irreversible.** Publishing, spending, and legal acceptance require a human click. Everything else is automated.
4. **Original by construction.** The system analyses competitors statistically and never passes competitor imagery into a generative prompt. Style is described in the abstract ("muted earth palette, condensed slab serif, centred badge lockup"), never by reference.
5. **Degrade, never die.** Any external provider failure produces a partial, clearly-labelled result rather than a failed run.
6. **Reproducible.** Any run can be replayed. Any score can be recomputed. Any prompt can be inspected with its exact inputs and model version.
7. **Speed is a feature.** Perceived latency is managed with streaming progress, partial results and optimistic UI. The operator must never stare at a spinner with no information.

## 7. Key product concepts (domain model in plain English)

| Concept | Definition |
|---|---|
| **Workspace** | The tenancy boundary. Owns all data. One in Phase 1. |
| **Niche** | A market the operator is investigating, e.g. "Gardening". Persistent across runs. |
| **Sub-niche** | A named segment within a niche, e.g. "Composting". Discovered by the system, rankable. |
| **Product Type** | A physical product category, e.g. "T-Shirt". Maps to Printify blueprints. |
| **Research Run** | One execution of the pipeline for a (niche, product type, style) triple. The unit of work, cost and audit. |
| **Competitor Shop / Listing** | Observed market entities with time-series snapshots. |
| **Style Profile** | The extracted visual fingerprint of a listing: palette, typography class, layout archetype, mockup style. |
| **Success Factor / Anti-Factor** | A statistically-qualified statement about what correlates with high/low performance in this niche. |
| **Market Gap** | A (sub-niche × angle × style) cell with high demand signal and low supply. |
| **Concept** | A named, described design idea with audience, style, reasoning and predicted scores. Text only. |
| **Artwork** | A rendered, processed, print-validated asset derived from a concept. |
| **Product Draft** | Concept + artwork + blueprint + variants + pricing + SEO, pre-publication. |
| **Listing** | A published (or draft) Etsy listing, tracked over time. |
| **Performance Snapshot** | Periodic capture of a listing's views, favourites, orders, revenue. |
| **Scoring Config** | A versioned set of weights and normalisation parameters. |

## 8. User-visible feature list (Phase 1)

### 8.1 Research
- New Research wizard: niche, product type, style preference (incl. *Auto Select Best Style*), depth (Quick / Standard / Deep), budget cap.
- Live run progress with per-step status, partial results, cost meter, cancel and resume.
- Run history with filters, re-run, clone-with-changes, and diff between two runs of the same niche.

### 8.2 Reports
- **Opportunity Report:** overall 0–100 score, five sub-scores with radar chart, verdict band (Avoid / Marginal / Good / Strong / Exceptional), ranked sub-niches with per-sub-niche scores, seasonality curve, evidence panel.
- **Competitor Report:** shop table (sortable/filterable), listing table with 20+ columns, price distribution, image-count distribution, review-velocity scatter, shop-age vs revenue plot, per-listing detail drawer with thumbnail and extracted style profile.
- **Success Report:** ranked weighted factors with support %, lift, sample size and confidence; palette gallery; typography breakdown; layout archetype breakdown; correlation matrix; "the median winning listing looks like this" synthesis card.
- **Failure Report:** ranked anti-factors, same statistical treatment, plus a side-by-side "Do / Avoid" comparison sheet.
- **Gap Report:** demand-vs-supply bubble map, ranked gaps with Gap Opportunity Score, and per-gap suggested angles.
- Export: PDF and CSV for every report.

### 8.3 Creation
- Concept board: 20 concepts (10 success-derived, 10 gap-derived) as cards with scores, reasoning, style tags; multi-select; regenerate individual or all; manual concept entry.
- Design Success Predictor panel per concept with five sub-scores and reasoning.
- Legal & Safety screen: per-concept risk level, flagged terms, matched marks with registry links, suggested safer alternatives, mandatory acknowledgement for anything above Low.
- Artwork Studio: brief viewer/editor, generation with variant count, side-by-side variant comparison, print-readiness QA panel (DPI, transparency, bleed, colour-count, edge quality), regenerate, upscale, background-remove, vectorise.

### 8.4 Commerce
- Product recommendation table: blueprint × print provider × variants, with demand/competition/profit sub-scores, cost, suggested price, margin.
- Pricing calculator with Etsy fee model, shipping profile, and margin target.
- SEO workspace: 10 listing variations, each with title (≤140 chars), description, 13 tags (≤20 chars each), keyword rationale, positioning statement; per-variation regeneration; character/quality validators; manual edit.
- Printify panel: upload, product creation, variant selection, mockup gallery.
- Etsy panel: draft creation, image ordering, attributes, inventory/pricing.
- **Final Review page**: everything on one screen with a profit estimate and a single Approve & Publish action behind a confirmation.

### 8.5 Analytics
- Portfolio dashboard: published products, revenue, views, conversion, best/worst performers.
- Prediction accuracy: predicted Design Success Score vs realised performance, with calibration curve.
- Niche performance comparison.
- Cost dashboard: spend per run, per product, per provider, month-to-date vs budget.
- Learning panel: current scoring config version, what changed, effect on historical scores.

## 9. Success metrics

### 9.1 Product outcome metrics (the ones that matter)
| Metric | Baseline (manual) | Target (6 months of use) |
|---|---|---|
| Revenue per published listing | — | +100% |
| Share of listings with ≥1 sale in 90 days | ~20% | ≥ 45% |
| Operator hours per published product | ~2.0 h | ≤ 0.15 h |
| Products published per week | 5–10 | 15–25 |
| Listings removed for IP issues | occasional | 0 |

### 9.2 System metrics
| Metric | Target |
|---|---|
| Opportunity Report p95 latency | < 4 min |
| Full run (Steps 1–8) p95 latency | < 11 min |
| Run success rate (no manual intervention) | > 97% |
| Draft-creation success rate | > 95% |
| Prediction calibration (Brier score, top-quartile classification) | < 0.20 after 200 outcomes |
| Cost per full research run | < £1.20 |

### 9.3 Leading indicators
Runs started per week · concepts accepted per run (target 3–6 of 20) · artwork accept rate (target > 55% first-pass) · SEO variations edited before publish (target < 40% — high edits mean the SEO engine is weak).

## 10. Assumptions

1. The operator holds a valid EverBee subscription and can export data, **or** accepts reduced fidelity using Etsy public data only.
2. Etsy Open API v3 access is granted for the operator's own shop (personal access, draft listing scopes).
3. Printify account exists with at least one connected Etsy shop.
4. Ideogram API access is available; if not, the artwork adapter is swappable.
5. The operator is the sole user and is trusted (no internal threat model in Phase 1).
6. Estimated sales/revenue figures from third-party tools are **estimates**, and the product must present them as such everywhere.

## 11. Constraints

| Constraint | Implication |
|---|---|
| Etsy API: 10,000 requests/day, 10 req/s | All Etsy calls go through a token-bucket limiter with a daily budget reserve for publishing. |
| Etsy: 13 tags max, 20 chars each; title 140 chars | Hard validators in the SEO engine, enforced before submission. |
| Printify: rate-limited, 200 req/30 min on some endpoints; publish is async | Queue-based with backoff; webhook-driven completion. |
| Ideogram: per-image cost and generation latency 10–40 s | Batch, cache, and gate behind concept approval. |
| EverBee: no documented public API | Provider abstraction with CSV as the guaranteed path (doc 11). |
| LLM context and cost | Aggregate before prompting; never send 400 raw listings to a model. |
| Single operator | No need for real-time collaboration, but no shortcuts on data model tenancy. |

## 12. Open product questions (to resolve before Phase 2 exit)

| # | Question | Owner | Needed by |
|---|---|---|---|
| Q1 | Does the operator's EverBee plan permit bulk export of shop-level listing data? Determines default acquisition adapter. | Operator | Phase 2 start |
| Q2 | Preferred default currency and VAT treatment for margin calculations. | Operator | Phase 4 |
| Q3 | Which Printify print providers are pre-approved (affects blueprint whitelist and cost table)? | Operator | Phase 4 |
| Q4 | Risk appetite threshold for the Legal Engine — does "Medium" block or warn? Default: warn + require acknowledgement. | Operator | Phase 3 |
| Q5 | Target margin floor for the pricing engine (default 40% after Etsy fees). | Operator | Phase 4 |
| Q6 | Should the system ever auto-publish (never, or after N successful manual approvals)? Default: never. | Operator | Phase 5 |
