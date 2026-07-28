# POD Intelligence — Phase 1D

# Master Implementation Roadmap

**Version:** 2.0 · **Status:** For execution
**Prerequisites:** [1A Product](PART-1-product-definition.md) · [1B Architecture](PHASE-1B-technical-architecture.md) · [1C AI Systems](PHASE-1C-ai-systems-and-integrations.md)

> **Purpose.** The definitive development roadmap from architecture to working product. Execution-focused: decisions, milestones and actions, not explanation. Previous phases are not redesigned here.

---

## Contents

| § | Section |
|---|---|
| 1 | [Development Strategy — V1 / V2 / V3](#1--development-strategy) |
| 2 | [Milestone Roadmap](#2--milestone-roadmap) |
| 3 | [Milestone 0 — Validation Before Building](#3--milestone-0--validation-before-building) |
| 4 | [Critical Path Analysis](#4--critical-path-analysis) |
| 5 | [AI Application Build Strategy](#5--ai-application-build-strategy) |
| 6 | [Data & Market Intelligence Implementation](#6--data--market-intelligence-implementation) |
| 7 | [Design Generation Implementation](#7--design-generation-implementation) |
| 8 | [Product Creation Workflow](#8--product-creation-workflow) |
| 9 | [UI/UX Dependencies](#9--uiux-dependencies) |
| 10 | [Timeline Estimates](#10--timeline-estimates) |
| 11 | [Development Rules](#11--development-rules) |
| 12 | [Executive Summary](#12--executive-summary) |

---

# 1 — Development Strategy

## 1.1 Three versions, three entry conditions

```
   V1 — PERSONAL MVP            V2 — ADVANCED AUTOMATION      V3 — SaaS PLATFORM
   ─────────────────            ────────────────────────      ──────────────────
   You, one shop.               You, less manual work.        Many users.
   Full loop working.           The loop improves itself.     Subscriptions.

   ENTRY: now                   ENTRY: V1 in daily use        ENTRY: V2 proven +
   EXIT: you use it daily       1 month, 30+ products         value articulable
                                published                     in one sentence
```

**The rule:** a version begins when the previous one is demonstrably working in daily use — not when it is "mostly done".

---

## 1.2 Version 1 — Personal MVP

**Definition:** a single-user tool that analyses markets, generates products, and creates Etsy/Printify drafts.

**The test:** *would you choose this over your current process, every day?*

| Area | In V1 |
|---|---|
| **Market data** | Etsy API · CSV import · manual entry · EverBee export |
| **Research** | Sub-niche discovery · opportunity scoring · seasonality · shop selection with fallback ladder · listing collection |
| **Visual analysis** | Colour measurement · typography, layout, mockup, subject classification · embeddings · content-hash caching |
| **Analysis** | Success · failure · market gap engines, complete with full statistical rigour |
| **Concepts** | 20 concepts (10 success-derived, 10 gap-derived) with citations and risk levels |
| **Scoring** | Six scores with contribution vectors and confidence levels |
| **Legal** | Complete screening with a service-layer blocking gate |
| **Artwork** | Brief · Ideogram · processing · the six-check evaluation gate |
| **Commerce** | Product recommendation · real-cost pricing with margin floor · 10 SEO variations |
| **Publishing** | Printify build · Etsy draft · final review · human-approved publish |
| **Tracking** | Daily performance sync · lineage · cost tracking |
| **Learning** | **Recording only.** Outcome records built from day one; fitting disabled. |

### Not in V1

| Excluded | Reason |
|---|---|
| Multi-user, teams, billing | One user. Nothing to sell yet. |
| Browser extension | Convenience layer; four providers already work |
| Active weight recalibration | Needs ~50 outcomes = 3–6 months of use |
| Bulk publishing | Encourages the volume strategy this product replaces |
| Additional marketplaces | Etsy mechanics are baked into the analysis |
| Public API, mobile | No consumer; desktop analysis tool |
| Automatic publishing | Never — a permanent principle |

---

## 1.3 Version 2 — Advanced Automation

**Definition:** same operator, materially less manual effort, materially better decisions.

**Priority order within V2 — the learning loop comes first**, because it is the compounding asset and every month delayed is outcome data not converted into better decisions.

| Sub-version | Contents |
|---|---|
| **V2.1 Learning** | Active weight recalibration · factor validation reporting · proxy calibration per niche · concept-origin analysis · profit-based optimisation |
| **V2.2 Effort reduction** | Browser extension · scheduled re-research with change alerts · batch operations with per-item review · draft review queue · automatic cost-drift repricing · saved run templates |
| **V2.3 Decision quality** | SEO A/B testing on live listings · run comparison over time · portfolio margin monitoring · order-level cost reconciliation · seasonal planning view |

**V2 exit:** at least one recalibration proposed, back-tested and activated; scores show measurable correlation with realised outcomes; profit per listing above the pre-system baseline.

---

## 1.4 Version 3 — SaaS Platform

**Definition:** multi-user product sold by subscription, or saleable as an asset.

| Sub-version | Contents |
|---|---|
| **V3.1 Multi-tenancy** | Enable isolation enforcement · invitations and roles · per-workspace credentials · fair scheduling |
| **V3.2 Commercial** | Subscription billing **with usage components** · usage metering · plan entitlements · onboarding for non-technical users · guided provider setup |
| **V3.3 Platform** | Shared market-fact caching · cross-account benchmarking · public API · white-label reports |

**Already architected:** every table carries a workspace identifier, isolation policies are written and tested, credentials are per-workspace encrypted, cost is tracked per call per workspace. V3.1 is largely switching things on.

**Do not start V3 before its entry condition.** Building billing before the product is proven is the most reliable way to spend months on infrastructure nobody wants.

---

## 1.5 Version comparison

| | V1 | V2 | V3 |
|---|---|---|---|
| Users | 1 | 1 | Many |
| Goal | The loop works | The loop improves itself | The loop sells |
| Learning | Recording only | **Active** | Cross-account |
| Legal engine | Complete | Refined | Same |
| Evaluation gate | Complete | Refined | Same |
| Billing | None | None | Full |
| Effort | ~21 weeks | ~10 weeks | ~10 weeks |

---

# 2 — Milestone Roadmap

Eleven milestones. Each states goal, rationale, features, components, dependencies, effort, completion criteria, risks, and — importantly — **what should not be built yet**.

---

## M0 — Thesis Validation

| | |
|---|---|
| **Goal** | Prove the market intelligence system discovers useful, non-obvious insights |
| **Why it exists** | The product rests on one unproven assumption. If it is false, every later milestone is wasted work. Testable in a week with throwaway scripts. |
| **Features built** | None. Throwaway analysis only. |
| **Technical components** | Spreadsheets and ad-hoc scripts. No production code. |
| **Dependencies** | None |
| **Effort** | 5 person-days |
| **Completion criteria** | See [§3](#3--milestone-0--validation-before-building) for the full pass/fail design |
| **Risks** | Operator marks most findings "already knew"; sample too small for significance |
| **What could go wrong** | Temptation to skip it and start building. Do not. |
| **Do not build yet** | Anything. This milestone produces a decision, not software. |

---

## M1 — Walking Skeleton

| | |
|---|---|
| **Goal** | Every architectural seam exercised and demonstrated, all providers faked |
| **Why it exists** | No feature milestone should discover that the orchestrator does not resume. Solve the hard infrastructure problems once, before they are load-bearing. |
| **Features built** | Auth · run creation · live progress · cancel · resume · budget enforcement · UI shell |
| **Technical components** | Monorepo · dependency-boundary enforcement · full database schema with partitioning and isolation policies · repository layer · seed fixture · orchestrator (step graph, runner, budget guard, heartbeats, reaper) · queue abstraction · observability · **fixture-backed fakes for every provider** · API middleware · progress streaming |
| **Dependencies** | M0 passed |
| **Effort** | 10 person-days |
| **Completion criteria** | Run starts, streams, completes · **worker killed mid-run resumes with identical output** · cancellation preserves completed steps · budget exhaustion pauses · progress stream reconnects · **full pipeline runs offline at zero cost** · seed data renders on every screen · boundary violations fail the build |
| **Risks** | Under-investment because it demos as a progress bar |
| **What could go wrong** | Starting on reports instead. Every later milestone then inherits unreliability that surfaces as intermittent, unreproducible failures. |
| **Do not build yet** | Any real provider adapter · any analysis logic · any report content beyond layout |

---

## M2 — Market Data Provider Layer

| | |
|---|---|
| **Goal** | Real competitor data flowing through a provider-agnostic layer |
| **Why it exists** | Everything downstream analyses this data. The layer must be modular from the start or EverBee dependency becomes structural. |
| **Features built** | Provider connection · shop discovery and selection · listing collection · CSV import with mapping · manual entry · competitor report |
| **Technical components** | **Normalised data contract** · provider interface with capability declaration · chain resolution · field-level merge with provenance · **Etsy API adapter** · **CSV import adapter** · **manual entry** · EverBee export adapter · selection scoring with age multiplier and fallback ladder · immutable snapshots · asset capture with content hashing · degradation banners |
| **Dependencies** | M1 |
| **Effort** | 10 person-days |
| **Completion criteria** | Real data collected using **Etsy API alone** · same run works with CSV added, provenance visible · manual entries merge correctly · each provider disabled in turn degrades gracefully · fallback ladder visible · truncation reported · every value shows source and estimate status |
| **Risks** | Data volume insufficient for significance; Etsy-only path under-tested |
| **What could go wrong** | Developing with rich imported data throughout, so the degraded path fails on first real use |
| **Do not build yet** | Browser extension · visual analysis · any analysis engine |

---

## M3 — Visual Analysis

| | |
|---|---|
| **Goal** | Structured visual attributes with measured accuracy |
| **Why it exists** | The product's differentiator. Every visual finding depends on classification being correct. |
| **Features built** | Palette extraction · style classification · visual attributes on listing detail |
| **Technical components** | Perceptual-space colour clustering (deterministic) · palette family mapping · vision classification with constrained vocabularies · **content-hash caching** · batching · embeddings · **200-image golden set** · accuracy evaluation harness |
| **Dependencies** | M2 (needs images) |
| **Effort** | 7 person-days |
| **Completion criteria** | Accuracy meets thresholds on the golden set · colour extraction reproducible · cache hit rate meets target on repeat run · 400 listings within latency and cost budget · out-of-vocabulary output rejected not coerced · exclusion count displayed |
| **Risks** | Classification accuracy below threshold |
| **What could go wrong** | Proceeding to M4 with weak accuracy — produces confident findings about attributes never really measured |
| **Do not build yet** | Analysis engines · anything that consumes these attributes statistically |

---

## M4 — Analysis Engines ★ HIGHEST VALUE

| | |
|---|---|
| **Goal** | Produce the four reports: opportunity, success, failure, gaps |
| **Why it exists** | **This is the product.** Everything before is enabling; everything after is exploitation. |
| **Features built** | Opportunity report with five sub-scores and verdict · sub-niche ranking · seasonality · success findings with baselines and lift · synthesis card · failure findings with causality labels · Do/Avoid sheet · market gaps with demand floor · coverage heat map |
| **Technical components** | **Statistical primitives** (cohorting, contingency, effect size, optimal band, confidence, suppression, cross-shop representation check, factor weighting, correlation, interaction) · success engine · failure engine · sub-niche discovery · opportunity scoring · coverage matrix · gap scoring with feasibility · report UIs with evidence drill-through |
| **Dependencies** | M2 + M3 |
| **Effort** | 15 person-days — **do not compress** |
| **Completion criteria** | All four reports within budget · **every finding shows cohort support, baseline, lift, n, confidence** · sub-threshold findings separated · single-shop findings suppressed · demand floor excludes with reasons retained · evidence drill-through resolves · **scores reproducible** · synthesis card usable as a brief |
| **Risks** | Findings weak or obvious at production quality; multiple-comparison noise |
| **What could go wrong** | The M4 checkpoint (re-run the M0 test on a fresh niche) fails. **Stop and diagnose — do not build on top.** |
| **Do not build yet** | Concept generation · anything creative |

---

## M5 — Concepts & Opportunity Scoring

| | |
|---|---|
| **Goal** | 20 grounded concepts with six scores each |
| **Why it exists** | Bridges research to creation — the link no competitor tool has |
| **Features built** | Concept board · concept detail with citations · six scores with contribution breakdown · regeneration with steering · manual concept entry |
| **Technical components** | **Grounding assembly** (statistics only, verified free of competitor identifiers) · parallel success and gap generation · output contracts with repair · embedding · within-set and history deduplication · quota refill · **groundedness evaluation** · category weight configuration · Market Fit · Originality · Competition · provisional Visual Quality and POD Suitability · **Estimated Potential as weighted sum × suitability factor** · contribution vectors · reasoning rendered from contributions · confidence levels |
| **Dependencies** | M4 |
| **Effort** | 10 person-days |
| **Completion criteria** | 20 verifiably distinct concepts · every citation resolves and matches its stored statistic · **no competitor identifier anywhere in the generation path** · scores reproducible · reasoning matches score exactly · below-threshold evidence shows no score · **no interface surface uses prediction or guarantee language** |
| **Risks** | Concepts read as generic pastiche |
| **What could go wrong** | Operator selects fewer than three of twenty — grounding context too thin or too abstract |
| **Do not build yet** | Artwork generation · legal screening beyond the interface stub |

---

## M6 — Legal Gate ★ NON-DEFERRABLE

| | |
|---|---|
| **Goal** | Screening that cannot be bypassed |
| **Why it exists** | Guards the only risk that can end the business. Far harder to retrofit into a system already generating artwork. |
| **Features built** | Screening results with matched registrations · risk levels · acknowledgement and override flows · safer alternatives · burned-terms list |
| **Technical components** | Entity extraction · normalisation · internal blocklist · **registry adapters** across jurisdictions and relevant goods classes · caching · **explicit reduced-coverage reporting on registry failure** · marketplace policy terms · copyright assessment · **deterministic risk rule table** · safer alternative generation with auto re-screening · **service-layer gate** · override with step-up auth and audit · adversarial evaluation set |
| **Dependencies** | M5 |
| **Effort** | 7 person-days |
| **Completion criteria** | **Direct service call with a blocked concept fails and makes zero provider calls** · adversarial recall meets the release gate · rule table deterministic · registry failure surfaces reduced coverage, never a clean pass · high-risk override requires re-auth, typed confirmation, justification · **blocked has no override path** |
| **Risks** | Registry coverage gaps; false-positive rate high enough to be ignored |
| **What could go wrong** | The gate is enforced only in the UI. Test by calling the service directly. |
| **Do not build yet** | Artwork generation |

---

## M7 — Artwork Pipeline

| | |
|---|---|
| **Goal** | Print-ready artwork through an automated evaluation gate |
| **Why it exists** | Without the gate there is no automation, only faster mistakes |
| **Features built** | Brief viewer and editor · variant generation · artwork studio · quality panel with remedies · originality panel · downloads |
| **Technical components** | Deterministic preparation (palette derivation, dimensions, constraints) · brief authoring · **prompt compilation in the adapter** · style templates · Ideogram adapter with provider rewriting disabled · independent variant calls with deterministic seeds · background removal with edge refinement · auto-crop · upscale with excessive-factor guard · renditions · vectorisation · **the six-check evaluation gate** (print quality, **text accuracy**, visual quality, POD suitability, similarity and copyright, **market attribute match**) · remediation decision tree · **diagnostic regeneration** |
| **Dependencies** | M6 (gate must exist first) |
| **Effort** | 12 person-days |
| **Completion criteria** | Four variants within cost budget including retries · **artwork failing a blocking check cannot attach to a product** · **induced misspelling is caught** · attribute drift detected and reflected in artwork-stage Market Fit · each failure names its specific remedy · diagnostic regeneration amends the brief · retries capped with a diagnosis beyond the cap |
| **Risks** | Output quality below commercial standard; text rendering unreliable |
| **What could go wrong** | Skipping text accuracy because it seems fussy — until the first misspelled shirt reaches a customer |
| **Do not build yet** | Product configuration · pricing · SEO |

---

## M8 — Commerce

| | |
|---|---|
| **Goal** | Product configuration, real-cost pricing, listing content |
| **Why it exists** | Converts artwork into a sellable, profitable product |
| **Features built** | Ranked product configurations · variant selection · pricing panel with itemised waterfall · 10 SEO variations with evidence · listing editor with live validation |
| **Technical components** | Catalogue sync (structures weekly, **costs daily**) · **cost drift monitoring with auto-repricing** · product recommendation scoring · colour recommendation validated against artwork ink · **full fee stack with advertising modelled as charged** · price solving · **margin floor blocking** · competitor price context · keyword pool with **sales weighting** · ten positioning axes · **hard constraint validation with repair** · quality scoring with competitive fit targeting moderate · text screening |
| **Dependencies** | M7 |
| **Effort** | 10 person-days |
| **Completion criteria** | Ranked configurations with real costs and correct margins · **both advertising cases shown, floor evaluated against the pessimistic one** · below-floor blocked with required price stated · colour incompatibility flagged · cost change triggers repricing and alert · ten demonstrably distinct variations · every keyword shows evidence · validation 100% after repair |
| **Risks** | Fee model inaccurate; catalogue costs stale |
| **What could go wrong** | Optimistic margin arithmetic reaching production |
| **Do not build yet** | Publishing |

---

## M9 — Publishing

| | |
|---|---|
| **Goal** | Product published to a live listing, with zero duplicates |
| **Why it exists** | Removes 45 minutes of manual work per product — the most immediate time saving |
| **Features built** | Printify build panel · placement editor · mockup gallery with ordering · Etsy panel · **final review page with checklist** · publish with confirmation |
| **Technical components** | Artwork upload with hash dedup · placement computation · idempotent product creation with reconciliation · **asynchronous mockup retrieval, never blocking** · mockup filtering and ordering · **draft-only creation** · ordered image upload with per-image keys · inventory with SKU mapping · **mandatory marketplace linking** · **server-side pre-flight re-evaluating every hard check** · **four-layer duplicate prevention** · targeted per-operation retry |
| **Dependencies** | M8 |
| **Effort** | 10 person-days |
| **Completion criteria** | Product publishes end to end · **zero duplicates under chaos testing** · reconciliation adopts rather than creates · publishing refused server-side when any hard check fails **even when the request claims otherwise** · partial failure repairs by targeted retry · authorisation loss pauses resumably · missing marketplace link blocks · operator attention under target |
| **Risks** | Duplicate listings; silent fulfilment-link omission |
| **What could go wrong** | A duplicate on a live shop — the worst operational failure this system can cause |
| **Do not build yet** | Analytics dashboards · learning fitting |

---

## M10 — Tracking & Hardening

| | |
|---|---|
| **Goal** | Outcomes flowing, system production-solid |
| **Why it exists** | **Outcome data cannot be recovered retrospectively.** Every month not recorded is permanently lost. |
| **Features built** | Performance dashboard · per-niche comparison · concept-origin comparison · accuracy view · cost dashboard · learning readiness · notifications · export |
| **Technical components** | Performance sync (daily, more frequent first week) capturing **views, clicks, favourites, orders, revenue, state** · immutable snapshots with deltas at write · derived metrics · nightly percentiles · drift detection · **outcome record building** · profit calculation from real costs · lineage assembly · dashboards · restore drill · runbooks · load and security testing |
| **Dependencies** | M9 |
| **Effort** | 10 person-days |
| **Completion criteria** | Performance data flowing daily · **outcome records verified by inspection** · any listing traces fully to its research · accuracy view populates · learning readiness shows a specific number · cost tracking accurate against invoices · restore drill recorded · every alert has a runbook |
| **Risks** | Outcome recording subtly wrong and unnoticed for months |
| **What could go wrong** | Deferring outcome recording to V2. The data is gone. |
| **Do not build yet** | Weight fitting · multi-user · billing |

---

## Roadmap summary

| M | Name | Days | Cumulative |
|---|---|---|---|
| 0 | Thesis validation | 5 | 5 |
| 1 | Walking skeleton | 10 | 15 |
| 2 | Market data layer | 10 | 25 |
| 3 | Visual analysis | 7 | 32 |
| 4 | **Analysis engines** | **15** | 47 |
| 5 | Concepts & scoring | 10 | 57 |
| 6 | Legal gate | 7 | 64 |
| 7 | Artwork pipeline | 12 | 76 |
| 8 | Commerce | 10 | 86 |
| 9 | Publishing | 10 | 96 |
| 10 | Tracking & hardening | 10 | **106** |

---

# 3 — Milestone 0 — Validation Before Building

## 3.1 The assumption under test

> **Does the market intelligence system discover useful, non-obvious insights that improve product decisions?**

Three ways it can fail, each fatal, each cheap to test now and expensive to discover at M4:

| Failure | Consequence |
|---|---|
| Findings are **obvious** | The research layer adds no value; the product is a publishing tool |
| Findings are **noise** | Actively harmful — confident wrong advice |
| Findings are **unactionable** | Interesting, but changes nothing about what gets made |

---

## 3.2 Test design

| Parameter | Specification |
|---|---|
| **Niches** | **3** — one you believe you understand well, one you are unsure about, one you have never worked in |
| **Listings per niche** | **150 minimum, 200 target** — below 150, cohorts are too small for significance |
| **Shops per niche** | 8–12, selected by the same criteria the system will use |
| **Cohort split** | Top decile (~15–20 listings) versus full population |
| **Data source** | Whatever is fastest — manual export, spreadsheet, hand collection. **No infrastructure.** |
| **Duration** | 5 working days total |

### The third niche matters most

The unfamiliar niche is the real test. In a niche you know well, "already knew" answers are expected and prove little. In one you have never worked in, the system must produce findings that are *checkable* and *plausible to an expert* — and you can validate them by spot-checking listings afterwards.

---

## 3.3 Data to collect per listing

| Category | Fields | Effort |
|---|---|---|
| **Commercial** | Price · shipping cost · free shipping · sales estimate · review count · review velocity · listing age · shop age · bestseller flag | Export or observation |
| **Content** | Title · tags · tag count · description length · image count | Export |
| **Visual** | Palette family · dominant colour · typography class · layout archetype · mockup style · complexity band · text presence | **Manual or ad-hoc classification** |

**Visual attributes are the expensive part and the essential part.** They are the differentiator. Classify by eye against the defined vocabularies from 1C §4.5 — accuracy matters more than speed here, and doing it by hand also validates that the vocabularies are usable.

---

## 3.4 Analysis to perform

For each niche, compute in a spreadsheet or throwaway script:

```
   1. Age-normalised performance = sales estimate ÷ min(age in months, 12)
   2. Assign cohorts: top decile, top quartile, bottom quartile
   3. For every categorical attribute:
        cohort support · baseline support · LIFT · sample size · significance
   4. For every numeric attribute:
        cohort median vs baseline median · optimal band
   5. Suppress: n < 8, or p > 0.10, or present in fewer than 3 shops
   6. Rank surviving findings by lift × confidence × sample
   7. Produce a findings list per niche
```

**Also compute the failure cohort** — the same treatment on bottom-quartile listings aged 90+ days. Failure findings are half the value and are the ones most likely to be surprising.

---

## 3.5 Evaluation method — blind review

**Order matters. Do not look at the statistics before answering.**

```
   ① Produce the findings list — statements only, NO numbers attached
        e.g. "Top performers use muted green palettes"
             "Top performers use 8 or more images"
             "Under-performers price below £16"

   ② Shuffle the list, including 3–5 DELIBERATELY FALSE findings as controls

   ③ Mark each one, before seeing any statistic:

        ALREADY KNEW    I would have said this without the analysis
        SUSPECTED       I had a hunch but no confidence
        SURPRISING      I did not expect this
        DISAGREE        I believe this is wrong

   ④ Then reveal the statistics and review

   ⑤ For each SURPRISING and DISAGREE finding, spot-check 5 listings by hand
```

### Why the false controls

If you mark planted false findings as "already knew", the review is not discriminating and the whole test is invalid. **The controls validate the reviewer, not the system.**

Target: **at least 4 of 5 false findings marked DISAGREE.** Below that, redo the review with more care.

---

## 3.6 Pass / fail criteria

| # | Criterion | Threshold |
|---|---|---|
| **1** | **Non-obvious rate** | **≥ 30%** of surviving findings marked SURPRISING or SUSPECTED |
| **2** | **Actionability** | You can name **≥ 3 specific design decisions** you would make differently |
| **3** | **Control discrimination** | ≥ 4 of 5 false controls correctly marked DISAGREE |
| **4** | **Cross-niche coherence** | The method produces coherent findings in **all three** niches, including the unfamiliar one |
| **5** | **Spot-check survival** | ≥ 70% of SURPRISING findings hold up on manual inspection |
| **6** | **No absurdities** | Zero findings you judge obviously wrong after seeing the evidence |
| **7** | **Sample viability** | ≥ 10 findings survive suppression thresholds per niche |

**PASS = all seven.** Anything less is a conditional pass at best and requires a documented decision.

---

## 3.7 If validation succeeds

Proceed to M1 unchanged, and **record two things**:

| Record | Use |
|---|---|
| The findings that were most surprising | These become the demo content and the marketing proof |
| The suppression thresholds that produced clean results | These become the initial production configuration, rather than guessed defaults |

---

## 3.8 If validation fails — what changes

| Failure mode | What it means | What changes in the product |
|---|---|---|
| **Non-obvious rate below 30%** | The analysis confirms rather than reveals | **Reposition.** Lead with market gaps and publishing automation. Demote success analysis to supporting evidence. The product becomes an execution tool with light research, not a research tool. |
| **Findings unactionable** | Right analysis, wrong output format | **Rework the synthesis layer.** The findings list may be correct but the presentation is not translating into decisions. Strengthen the synthesis card; weaken the raw finding list. |
| **Findings noisy — spot-checks fail** | Statistical thresholds too loose | Raise thresholds, increase collection volume to 300+ per niche, re-test. **If still noisy at higher volume, the visual attribute vocabularies are probably too coarse** — refine them and re-test. |
| **Fewer than 10 findings survive per niche** | Insufficient data at realistic volumes | Increase per-shop and per-run caps; drop the Quick depth option; set expectations that only Standard and Deep produce reliable analysis. |
| **Incoherent in the unfamiliar niche** | The method depends on operator context to interpret | **Serious.** The product cannot claim to work in niches the user does not already know — which removes most of its value. Re-examine whether visual classification is niche-dependent. |
| **Controls not discriminated** | The review was not rigorous | Redo the review. This is a measurement failure, not a product failure. |

**In every case: fix and re-test before building.** A week repeated is cheap. Five months on a false premise is not.

---

# 4 — Critical Path Analysis

## 4.1 The sequence

```
   FOUNDATION                    M1   infrastructure, orchestrator, fakes
        ↓
   DATABASE                      M1   schema, repositories, seed
        ↓
   MARKET DATA PROVIDER LAYER    M2   normalised contract, 4 providers
        ↓
   DATA PROCESSING               M2/M3 collection, snapshots, visual analysis
        ↓
   ANALYSIS ENGINE               M4   ★ success, failure, gap
        ↓
   OPPORTUNITY SCORING           M4/M5 niche verdict, then design scores
        ↓
   AI ORCHESTRATION              M5   grounding, generation, contracts
        ↓
   DESIGN GENERATION             M5   concepts
        ↓
   DESIGN EVALUATION             M6/M7 legal gate, then artwork gate
        ↓
   SEO GENERATION                M8   listing content
        ↓
   PRINTIFY INTEGRATION          M9   product build
        ↓
   ETSY DRAFT CREATION           M9   draft listing
        ↓
   APPROVAL WORKFLOW             M9   review page, publish
        ↓
   LEARNING SYSTEM               M10  recording (fitting deferred to V2)
```

## 4.2 Why this order is correct

| Transition | Reason |
|---|---|
| Foundation → Database | The orchestrator writes step state; it cannot exist before the schema |
| Database → Provider layer | Providers write immutable observations; the shape must exist first |
| Provider layer → Processing | Nothing to process without data |
| Processing → **Analysis** | The analysis is the value. It requires both commercial and visual data. |
| Analysis → Opportunity scoring | Scores are computed from findings; findings must exist first |
| Scoring → AI orchestration | Grounding context is assembled from findings and scores |
| Orchestration → Generation | Generation consumes the grounding |
| Generation → **Evaluation** | Nothing to evaluate before something is generated — **but the legal gate must exist before artwork generation, or it will be retrofitted badly** |
| Evaluation → SEO | SEO describes an approved design |
| SEO → Printify | The product needs artwork and content |
| Printify → Etsy | Mockups come from Printify and become listing images |
| Etsy → Approval | Cannot approve what does not exist as a draft |
| Approval → Learning | Outcomes only exist after publishing |

**The one inversion worth noting:** the legal gate (M6) sits *before* artwork generation (M7) even though it evaluates concepts that already exist. This is deliberate — building generation first and adding the gate afterwards means auditing every call path retrospectively, and the gate will leak.

---

## 4.3 The four answers

### Highest-value milestone: **M4 — Analysis Engines**

Everything before is enabling. Everything after is exploitation of what M4 produces. If M4 works, the product has a reason to exist even if nothing after it is built — a research tool that reveals non-obvious market structure is independently valuable.

**Implication:** budget it generously, do not compress it, and re-run the M0 test at its exit.

### Biggest technical risk: **duplicate listings and irreversible marketplace actions (M9)**

Every other technical failure is recoverable. A duplicate listing on a live shop, an accidental publish, or a product that sells but never manufactures is externally visible, damages reputation, and requires manual remediation.

**Mitigation:** four-layer duplicate prevention, draft-only automation, server-side pre-flight, mandatory fulfilment linking, and chaos testing as a milestone gate.

**Runner-up:** visual classification accuracy (M3). If it is weak, M4 produces confident findings about attributes never really measured — and nothing downstream would reveal the problem.

### Biggest business risk: **the analysis confirms rather than reveals**

If findings are things you already knew, the research layer adds no value and the product collapses to a publishing automation tool — which is useful but far less valuable and far more replaceable.

**Mitigation:** M0 tests this before any code exists. The M4 checkpoint tests it again at production quality.

### Most common way this project fails

**Building horizontally instead of vertically, then running out of momentum before anything works end to end.**

The specific pattern: months spent on a beautiful data model and comprehensive API surface, with no working pipeline, no report anyone has read, and no product to use. Enthusiasm decays, the project is shelved at 70% complete, and none of it has produced a single published listing.

**Mitigations:** M1 forces an end-to-end skeleton in week 3. Every milestone has a demo. The M4 checkpoint forces an honest assessment at the halfway point. And the cut list in §11.4 exists so scope can be reduced without abandoning the build.

**Second most common:** deferring outcome recording to "later". The learning loop is the moat, and its input data cannot be reconstructed retrospectively.

---

# 5 — AI Application Build Strategy

## 5.1 The governing principle

> **Claude is the reasoning layer, not the calculation layer.**

No number that appears anywhere in the product is produced by a model.

| Claude does | Deterministic code does |
|---|---|
| Understand, classify, create, explain | Every calculation, every threshold, every decision |

---

## 5.2 Build order for AI components

```
   1. AI GATEWAY        the single path to any model — build first
   2. PROMPT REGISTRY   versioned files with contracts
   3. EVALUATION HARNESS before any prompt is trusted
   4. CLASSIFICATION    M3 — simplest, highest volume, most cacheable
   5. EXTRACTION        M6 — legal entities
   6. GENERATION        M5, M7, M8 — concepts, briefs, listings
   7. EXPLANATION       M4 — summaries, last because they are least critical
```

**The gateway comes first.** Six engines need model calls. Without a single gateway, retry, caching, budgeting and validation are reimplemented six times, differently, with six places for a cost leak.

---

## 5.3 The gateway — what it owns

| Concern | Implementation note |
|---|---|
| Tier resolution | Capability tier → model, from configuration. **Never a model name in application code.** |
| Prompt rendering | From the versioned registry |
| **Untrusted content quarantine** | Delimited blocks, appended last, sanitised |
| Cache lookup | By input hash, deterministic purposes only |
| Budget reservation | **Before** the call |
| Rate limiting | Per tier, distributed |
| Contract enforcement | Validate → one repair → one stricter retry → explicit failure |
| Cost recording | Tokens, cost, latency, cache status, contract validity |
| Circuit breaking | Per tier |

---

## 5.4 Prompt management

| Rule | Consequence |
|---|---|
| Prompts are **files**, never inline strings | Reviewable, versionable, testable |
| Semantic versioning | Output-shape changes are major |
| Every generated entity records its prompt version | A prompt change never retroactively alters stored output |
| Registry validated at boot | A missing contract fails startup, not production |
| **Every prompt has an evaluation suite** | An AI feature without an evaluation is not shippable |

**Directory shape:** one folder per prompt id, one subfolder per version, containing the template, the output contract, metadata (tier, temperature, cache policy) and evaluation cases.

---

## 5.5 AI task separation — the eighteen jobs

| Category | Jobs | Tier |
|---|---|---|
| **Understanding** | Sub-niche discovery · legal entity extraction · copyright assessment · failure causality · gap feasibility | Analysis / Reasoning |
| **Classification** | Typography · layout · mockup · subject and humour · artwork safety | Vision |
| **Creation** | Success concepts · gap concepts · concept expansion · artwork briefs · listing content | Reasoning / Analysis |
| **Explanation** | Executive summary · gap explanations · safer alternatives | Analysis |

**No other model calls exist.** If a new one is proposed, it must be added to this list with a stated tier and an evaluation suite.

---

## 5.6 Deterministic scoring — build sequence

Build the pure functions **before** the engines that use them.

```
   packages/domain — no input, no output, no clock, no randomness
   ────────────────────────────────────────────────────────────
   1. normalisation and banding          (foundational)
   2. statistical primitives             (cohorts, lift, significance, effect size)
   3. confidence assignment
   4. factor weighting
   5. opportunity sub-scores
   6. gap scoring with demand floor
   7. the six design scores
   8. pricing and fee stack
   9. quality criteria evaluation
   10. legal risk rule table
```

**These are the highest-tested code in the system** — exhaustive unit tests plus property tests for monotonicity, bounds and invariance. They run in milliseconds with no network.

---

## 5.7 Data pipelines

```
   EXTERNAL DATA (untrusted)
        ↓
   PERCEPTION      classify + measure → structured records
        ↓
   AGGREGATION     ★ NO MODEL. All raw external text destroyed here. ★
        ↓
   SYNTHESIS       model receives ONLY statistics
        ↓
   EVALUATION      ★ NO MODEL. Scores, gates, validation. ★
        ↓
   HUMAN DECISION
```

**The aggregation boundary is load-bearing.** It cuts token cost ~85%, improves output quality, and makes prompt injection structurally ineffective — because injected text in a competitor listing never reaches the generative stage.

---

## 5.8 Error handling

| Failure | Behaviour |
|---|---|
| Contract invalid after repair and retry | Step fails; raw output preserved and viewable; retry available |
| Model declines | Recorded. **For legal prompts this escalates risk, never retries around.** |
| Timeout | Retry same tier → escalate tier → fail |
| Rate limited | Backoff honouring provider hint; step stays running with progress messaging |
| Provider outage | Circuit breaker opens; steps become `blocked_external`; run resumes automatically |
| Cost anomaly | Alert but allow completion — killing mid-flight wastes spend already incurred |
| Out-of-vocabulary output | **Rejected, never coerced** |

---

## 5.9 Human approval points

**Three, all mandatory, none removable.**

| Gate | Where | Why |
|---|---|---|
| **Concept selection** | After M5 | Cheap text before expensive images. Eliminates ~75% of image spend. |
| **Legal clearance** | After M6 | Accepting legal risk is a human decision |
| **Publish approval** | After M9 | Irreversible and public |

**No fourth gate is added, and none of the three is ever bypassed** — including for batch operations in V2, which get one confirmation per batch, not zero.

---

# 6 — Data & Market Intelligence Implementation

## 6.1 Build order

```
   1. NORMALISED CONTRACT     define before any provider
   2. PROVIDER INTERFACE      capability declaration, probe, fetch
   3. ETSY API PROVIDER       always available — build first
   4. CSV IMPORT PROVIDER     tool-agnostic
   5. MANUAL ENTRY            the floor beneath the floor
   6. MERGE + PROVENANCE      field-level, by confidence
   7. EVERBEE EXPORT          one provider among several
   8. COLLECTION              discovery, selection, snapshots
   9. FEATURE EXTRACTION      visual + textual
   10. ANALYSIS ENGINES       success, failure, gap
   11. OPPORTUNITY SCORING
```

**Etsy API first, not EverBee.** It is the provider that always works, requires no subscription, and must be the best-tested path. Building EverBee first would leave the fallback under-exercised.

---

## 6.2 The normalised contract — build rule

**Define the contract before writing any provider.** Every provider maps into it; no provider-specific field survives the mapping.

| Required on a listing | Everything else |
|---|---|
| `title` · `shop_reference` · `price` | **Optional** |

**This is what makes Etsy-API-only a first-class path rather than a degraded one.** A provider supplying only measured public data satisfies the contract completely.

Every record carries a provenance envelope: provider, fetch time, estimate flag, confidence, per-field source, derivation method.

---

## 6.3 EverBee modularity — the acceptance test

> **Adding or removing a provider must require zero changes to any engine.**

Test it explicitly: disable EverBee entirely and confirm the full pipeline runs on Etsy API plus CSV plus manual entry, with correct degradation banners and confidence downgrades.

**If any engine breaks, the layer is not modular and must be fixed before M4.**

---

## 6.4 Competitor analysis build sequence

| Step | Component | Note |
|---|---|---|
| 1 | Shop discovery | Merge candidates from search, imports, prior runs, sub-niche expansion |
| 2 | Qualification gates | Exclusions recorded with reasons |
| 3 | Selection scoring | Weighted formula **× age multiplier favouring shops under 3 years** |
| 4 | Fallback ladder | 20 → 10 → 5 → proceed degraded |
| 5 | Listing collection | Relevance filter, caps, **truncation reported** |
| 6 | Snapshots | Immutable, dated, provenance-tagged |
| 7 | Asset capture | Content-hashed, deduplicated |

---

## 6.5 Feature extraction

| Extracted | Method | Build note |
|---|---|---|
| Colour palette, family, dominant, contrast, count | **Deterministic clustering** | Build first — free, reproducible, and the highest-weighted visual attribute |
| Typography, layout, mockup, illustration style, complexity, subject, humour | **Vision classification** | Constrained vocabularies; build **with** content-hash caching, not after |
| Embeddings | Model | For similarity and originality |
| Tag and phrase frequency, relevance, **sales weighting** | Deterministic | Sales weighting is what makes SEO evidence-based |

---

## 6.6 The three analysis engines

| Engine | Cohort | Distinctive component |
|---|---|---|
| **Success** | Top decile by age-normalised performance | Baseline always computed alongside cohort support; synthesis card |
| **Failure** | Bottom quartile, **aged 90+ days**, drawn from the **same successful shops** | Causality labelling; crowded-loser detection; findings become **hard validators** where controllable |
| **Gap** | Coverage cells below supply threshold | **Demand floor with excluded cells retained and displayed** |

**The failure engine's shop control matters.** Analysing under-performers within successful shops isolates what makes a *listing* fail when the seller demonstrably knows how to succeed.

---

## 6.7 Opportunity scoring

Five sub-scores → weighted composite → verdict band. All deterministic, all from stored features, all recording the weight version that produced them.

**Every sub-score has a documented fallback** for when its input is unavailable, with confidence dropped and the method named in the interface.

---

# 7 — Design Generation Implementation

## 7.1 Build sequence

```
   RESEARCH                 M4 output: findings, gaps, resolved style
        ↓
   SUCCESS ATTRIBUTES       weighted findings → grounding table
        ↓
   MARKET GAPS              ranked gaps with evidence → grounding table
        ↓
   DESIGN BRIEFS            M7: palette derived deterministically,
                                brief authored by model within bounds
        ↓
   IDEOGRAM GENERATION      M7: independent variant calls, deterministic seeds
        ↓
   QUALITY EVALUATION       M7: the six-check gate
        ↓
   USER SELECTION           human decision
        ↓
   PRINT-READY ARTWORK      renditions, vectorisation
```

---

## 7.2 The brief — deterministic preparation first

**Before any model call:**

| Prepared deterministically | Source |
|---|---|
| Palette (explicit values) | Winning palette family from M4 findings |
| Target dimensions | Product type print area |
| Standing negative constraints | Fixed list |
| Anti-findings as prohibitions | M4 failure engine |

**The model composes with the palette; it does not choose it.** This keeps the highest-weighted visual attribute data-derived rather than model-invented.

**Typography is specified by class, never by font name** — avoids licensing implications and models inventing fonts that do not exist.

---

## 7.3 Ideogram integration

| Setting | Value | Reason |
|---|---|---|
| Variants | 4 default | Real choice without excessive spend |
| Call structure | **One per variant** | One failure costs one variant |
| Seeds | Deterministic per brief + index | Reproduction and controlled iteration |
| **Provider prompt rewriting** | **Disabled** | Breaks reproducibility and defeats negative constraints |
| Prompt compilation | **In the adapter** | Provider-specific tuning in one file; swapping providers is one new compiler |

**Hard constraint:** no code path accepts an external image as input, except the artwork's own prior renditions for upscaling.

---

## 7.4 Transparent backgrounds and POD requirements

| Requirement | Implementation |
|---|---|
| **True alpha transparency** | Provider output → local segmentation → hosted fallback. First success wins, method recorded. |
| Edge refinement | Alpha threshold cleanup · halo decontamination · feather ≤ 1px · stray island removal |
| Verification | Genuine alpha channel · ≥ 3% fully transparent (below this, removal failed) · ≤ 8% semi-transparent (above this, soft-edged mess) |
| **300 DPI at true print size** | Upscale with an excessive-factor guard — beyond the guard, regenerate rather than upscale |
| Minimum stroke width | Measured at print size |
| Colour count | Screen-print practicality |
| Gamut | DTG printability |

---

## 7.5 SVG / vector workflow

| Aspect | Detail |
|---|---|
| **Applied to** | Typography-led and flat-illustration output |
| **Skipped for** | Textured or photographic output — with the reason recorded |
| Method | Palette quantisation → alpha-aware trace → path simplification tuned to preserve letterforms |
| **Verification** | Rasterise the vector at print size and compare against the source. Below the similarity threshold, discard the vector and mark `svg_ready = false`. |

**Vectorisation is optional output, never a blocker.** A design that will not vectorise cleanly is still publishable as raster.

---

## 7.6 The six-check evaluation gate

| # | Check | Type | Severity |
|---|---|---|---|
| 1 | Print quality | Measured | **Blocking** |
| 2 | **Text accuracy** | Transcribe + exact compare | **Blocking** |
| 3 | Visual quality | Measured composite | Threshold |
| 4 | POD suitability | Measured | **Blocking below floor** |
| 5 | Similarity + copyright | Embedding distance + vision review | **Blocking** |
| 6 | **Market attribute match** | Re-analyse with the same classifier used on competitors | Threshold |

**Check 2 is the one most likely to be skipped and the one that causes the most damage if omitted.** Zero tolerance — a single wrong character fails. Also verify *no* text where none was specified, because models add text unprompted.

**Check 6 closes the loop between specification and output.** If the brief specified a badge layout and the model rendered a centred stack, the design no longer matches the finding that justified it, and its concept-stage score is now wrong.

---

## 7.7 Regeneration process

```
   FAILURE
     ↓
   deterministic fix available?  →  YES: apply, re-check
     ↓ NO
   cause diagnosable?            →  YES: DIAGNOSTIC REGENERATION
     ↓ NO                                 brief amended from the specific
   plain regeneration, new seed             failure, then regenerate
     ↓
   automatic attempts exhausted? →  present to operator WITH A DIAGNOSIS
```

| Failure | Fix | Cost |
|---|---|---|
| Low resolution · poor transparency · halo · aspect · colour count · gamut | Deterministic reprocessing | Negligible |
| Text inaccurate (minor) | Regenerate, **same seed**, text emphasised | One generation |
| Text garbled | Regenerate, new seed, simplified text | One generation |
| Strokes too thin | Regenerate with bolder line-weight brief | One generation |
| One attribute deviation | Regenerate with that attribute emphasised | One generation |
| Several deviations | **Revise the brief**, then regenerate | Brief + generation |
| Similarity too high | Regenerate with differentiation instruction | One generation |

**Automatic attempts are capped.** Beyond the cap the operator decides, with a specific diagnosis rather than a generic failure.

---

## 7.8 Copyright and trademark safety

**Two screening points, both mandatory:**

| Point | Screens | Blocks |
|---|---|---|
| **Before generation** (M6) | Concept text | Artwork generation entirely |
| **After generation** (M7, check 5) | Rendered artwork | Attachment to a product |

The second exists because a clean concept can produce risky artwork — models generate logos, faces resembling real people, and marks nobody requested.

**Enforcement is at the service layer**, where the external call is made. A client bypassing the interface still cannot generate artwork for a blocked concept.

---

# 8 — Product Creation Workflow

## 8.1 Build sequence

```
   CATALOGUE SYNC          structures weekly, COSTS DAILY
        ↓
   PRODUCT RECOMMENDATION  demand 35% · competition 25% · profitability 40%
        ↓
   COLOUR RECOMMENDATION   empirical, validated against artwork ink
        ↓
   PRICING                 full fee stack, advertising modelled as charged
        ↓
   ARTWORK UPLOAD          content-hash deduplicated
        ↓
   PRODUCT CONFIGURATION   placement, variants
        ↓
   MOCKUP RETRIEVAL        asynchronous, never blocking
        ↓
   SEO SELECTION           one of ten variations
        ↓
   ETSY DRAFT              draft state always
        ↓
   FINAL REVIEW            checklist, server-side pre-flight
        ↓
   USER APPROVAL           ← the only path to publishing
```

---

## 8.2 How AI recommends products

**It does not.** Recommendation is deterministic; the model contributes explanatory prose only.

```
   Product Score = 0.35 × Demand
                 + 0.25 × (100 − Competition)
                 + 0.40 × Profitability          ← highest, deliberately
```

| Family | Recommendation drivers |
|---|---|
| **Shirts** | Garment weights and cuts among winners · observed colour distribution · cost against achievable price band · provider reliability |
| **Hoodies** | Same, **plus pocket compatibility with artwork placement** — a design behind a pocket seam is a defect |
| **Mugs** | Size distribution among winners · wrap area against artwork aspect ratio · handle-seam safe margins |
| **Posters** | Format distribution among winners · paper type against artwork style · frame-compatibility of standard sizes |

**Colour recommendation is validated, not just ranked.** Recommending the winning colour is useless if the design disappears on it — contrast and ink-coverage checks run against the actual artwork.

---

## 8.3 Pricing — the non-negotiable rules

| Rule | Implementation |
|---|---|
| **Advertising fees modelled as charged by default** | Both figures shown; **the floor is evaluated against the pessimistic one** |
| Margin floor blocks, not warns | Below-floor configurations cannot be selected; the required price is stated |
| Costs snapshotted at configuration time | Catalogue refreshes daily; the cost used for a decision must not drift |
| Cost drift monitored | Beyond threshold → reprice unpublished drafts automatically, alert the operator |

**Optimistic margin arithmetic is the most common way POD sellers lose money.** The system refuses to participate.

---

## 8.4 Printify integration

| Operation | Requirement |
|---|---|
| Artwork upload | Deduplicated by content hash — never uploaded twice |
| Placement | Computed per product type, operator-adjustable, persisted, reapplied on update |
| Product creation | **Idempotent**, with reconciliation adopting on ambiguous timeout |
| Mockups | Asynchronous with backoff, **never blocking a request thread** |
| Mockup selection | Filtered to a preferred set, ordered by niche performance |
| Update | Modifies the existing product, **never creates a second** |
| **Marketplace linking** | **Mandatory, tracked, a blocking pre-publish check** |

**The link is a blocking check** because omitting it produces a silent failure where the listing sells and nothing is manufactured.

---

## 8.5 Etsy draft generation

**Eight independently idempotent operations:**

```
   upload artwork → create product → fetch mockups → create DRAFT listing
   → upload images → set inventory → link fulfilment → publish
```

| Rule | Detail |
|---|---|
| **Draft always** | No code path creates an active listing |
| Per-operation keys | Partial failure repairs by targeted retry, never recreating |
| Image upload | Ordered, per-image keys, primary first |
| Inventory | Full replacement — idempotent by construction |
| SKUs | Map marketplace variants to fulfilment variants for order routing and cost reconciliation |

---

## 8.6 Final approval workflow

**One screen. Everything needed for the decision.**

| Hard checks — block publishing | Soft checks — warn |
|---|---|
| Legal cleared or overridden with record | Image count below recommended band |
| Print-readiness passed | Price outside competitive range |
| Originality passed or acknowledged | Description below adequate length |
| Margin at or above floor | |
| Title present and within limit | |
| Full tag allowance, each within limit | |
| Category and shipping profile set | |
| **Production partner declared** | |
| **Fulfilment product linked** | |

**Server-side pre-flight re-evaluates every hard check from current data on every publish call**, regardless of what the client believes. The disabled button is a convenience; this is the control.

**Publishing requires explicit confirmation** restating the shop, the listing, and that it becomes publicly visible.

---

# 9 — UI/UX Dependencies

*Detailed design is a separate exercise. This identifies what must exist and what information each stage requires.*

## 9.1 Required screens by milestone

| Milestone | Screens |
|---|---|
| **M1** | Login · second factor · app shell with navigation · settings/integrations · run monitor (skeleton) |
| **M2** | New run wizard · run history · competitor report (shops, listings, distributions) · listing detail · CSV import with mapping · manual entry grid |
| **M3** | Visual attributes on listing detail · palette display |
| **M4** | Opportunity report · success report · failure report · Do/Avoid sheet · gap report with bubble map · evidence drawer |
| **M5** | Concept board · concept detail with scores and citations |
| **M6** | Legal screening panel · override flow |
| **M7** | Artwork studio: brief editor · variant comparison · quality panel · originality panel · tools |
| **M8** | Product configuration table · pricing panel · SEO workspace with editor and preview |
| **M9** | Printify panel with placement editor · mockup gallery · Etsy panel · **final review page** |
| **M10** | Dashboard · performance analytics · accuracy view · cost dashboard · listing detail with lineage |

**~22 screens for V1.**

---

## 9.2 Required user flows

| Flow | Stages |
|---|---|
| **Setup** | Account → second factor → workspace → connect Etsy → connect Printify → connect Ideogram → choose data provider → set economics |
| **Research** | Enter niche → select product type → choose style and depth → confirm cost → watch progress → read reports → **decide: proceed / refine / reject / deepen** |
| **Creation** | Review concepts → select → review legal → clear or override → review brief → generate → compare variants → review quality → accept |
| **Commerce** | Choose configuration → select variants → set price → review margin → choose SEO variation → edit if needed |
| **Publish** | Build product → review mockups → order images → final review → check checklist → confirm → publish |
| **Learn** | View performance → compare estimated vs actual → review by niche and concept origin → start new research informed by results |

---

## 9.3 Dashboard requirements

**Answer one question in five seconds: *what should I do next?***

| Section | Contents |
|---|---|
| **Status strip** | Published listings (30d) · revenue (30d) · products in pipeline · month-to-date spend vs budget |
| **Active work** | Running runs with progress · concepts awaiting selection · artwork awaiting acceptance · drafts awaiting review — **each linking directly to the next action** |
| **Health** | Integration status · notifications |
| **Performance** | Revenue over time with publication markers · best and worst performers |
| **Intelligence** | Highest-scoring unexplored niches · unused market gaps |

---

## 9.4 Information required at each stage

| Stage | Operator must see | Must not have to look up |
|---|---|---|
| Run creation | Estimated cost and duration · data sources and their health · budget remaining | What each depth option actually changes |
| During run | Named step · specific activity · elapsed time · accumulated cost · partial results | Whether it is still working |
| Opportunity report | Score · verdict · five sub-scores with inputs · confidence · what is degraded | Why a sub-score is what it is |
| Success report | **Cohort support, market baseline, lift, sample size, confidence — on every finding** | The baseline (it must never require a click) |
| Gap report | Demand evidence · supply count · the demand floor drawn | Why a cell was excluded |
| Concept board | Score · band · confidence · origin · risk level · cited evidence | Which findings produced this concept |
| Legal panel | Risk level · matched terms · each registration with owner, number, class, jurisdiction · rationale | What specifically triggered the flag |
| Artwork studio | Per-criterion measured value, threshold, status, **and remedy** | What to do about a failure |
| Pricing | Every cost line itemised · **both advertising cases** · margin · floor status · required price if below | The true net profit |
| Final review | Everything, on one page · hard vs soft checks · what blocks and why | Anything requiring navigation |
| Analytics | Estimated vs actual · calibration · by niche · by concept origin · full lineage | Why a listing exists |

---

## 9.5 Cross-cutting UI rules

| Rule | Reason |
|---|---|
| **Cost shown before every spending action** | No surprises |
| **Estimated values visually distinct from measured** | A stated product principle |
| **Every score one click from its evidence** | A score you cannot interrogate is one you should not trust |
| **Degraded results visibly degraded, with the remedy named** | Honest partial results beat silent gaps |
| Determinate progress with named steps | Nine minutes of spinner is a failure |
| Errors state what, why, and next action | Raw provider errors never shown |
| **No prediction or guarantee language anywhere** | Per 1C §10.0 |

---

# 10 — Timeline Estimates

## 10.1 Three scenarios

| | **Aggressive** | **Realistic** | **Conservative** |
|---|---|---|---|
| **Assumes** | Very fast AI-assisted development, no blockers, focused full-time | One person building consistently, normal interruptions | Includes debugging, provider surprises, iteration on quality |
| **Effective days/week** | 5 | 3.5 | 2.5 |
| **Total** | **~14 weeks** | **~20 weeks** | **~30 weeks** |
| **Calendar** | ~3.5 months | ~5 months | ~7 months |

**Base effort: 106 person-days.** The variance is entirely in productive days per week, which is the honest variable.

---

## 10.2 Milestone timeline

| M | Milestone | Days | Aggressive | Realistic | Conservative |
|---|---|---|---|---|---|
| 0 | Validation | 5 | Week 1 | Week 1–2 | Week 1–2 |
| 1 | Walking skeleton | 10 | Week 3 | Week 4 | Week 6 |
| 2 | Market data | 10 | Week 5 | Week 7 | Week 10 |
| 3 | Visual analysis | 7 | Week 6 | Week 9 | Week 13 |
| 4 | **Analysis engines** | 15 | **Week 9** | **Week 13** | **Week 19** |
| 5 | Concepts & scoring | 10 | Week 11 | Week 16 | Week 23 |
| 6 | Legal gate | 7 | Week 12 | Week 18 | Week 26 |
| 7 | Artwork | 12 | Week 14 | Week 21 | Week 31 |
| 8 | Commerce | 10 | Week 16 | Week 24 | Week 35 |
| 9 | Publishing | 10 | Week 18 | Week 27 | Week 39 |
| 10 | Tracking | 10 | Week 20 | Week 30 | Week 43 |

*Weeks shown are milestone completion points. Conservative exceeds the headline figure because it assumes lower weekly throughput throughout.*

---

## 10.3 Capability by point in time

| After | You can | Value |
|---|---|---|
| **M0** | Know whether the product is worth building | **Decision** |
| **M1** | Run a fake pipeline reliably | None to the user |
| **M2** | Collect and browse real competitor data | Modest — a better research view |
| **M3** | See visual attributes across a market | Real — no other tool does this |
| **M4** | **Get all four reports on any niche** | **★ Substantial — usable as a research product on its own** |
| **M5** | Get 20 scored, evidence-grounded concepts | High — research becomes creative direction |
| **M6** | Screen concepts for legal risk | Risk reduction |
| **M7** | Generate print-ready artwork | **High — designs without a designer** |
| **M8** | Configure products and generate listings | High |
| **M9** | **Publish end to end** | **★ Complete — the full loop** |
| **M10** | Track outcomes and see accuracy | Compounding |

**The two inflection points are M4 and M9.** After M4 you have a research product worth using. After M9 you have the complete product.

---

## 10.4 Recommended plan

**Present the realistic figure with a buffer: ~20 weeks + 5 weeks contingency = 25 weeks to V1.**

| Contingency allocation | Weeks |
|---|---|
| Market data harder than expected (M2) | +1 |
| Classification accuracy below threshold (M3) | +1 |
| Analysis findings need iteration (M4) | +2 |
| Artwork quality needs brief iteration (M7) | +1 |

**Do not present the aggressive figure as the plan.** A plan that is beaten builds confidence; a plan that slips erodes it.

---

# 11 — Development Rules

## 11.1 Prioritise

| Rule | In practice |
|---|---|
| **Working product over perfection** | Ship the milestone when it meets its criteria, not when it is elegant. Refactor when a second use case appears, not in anticipation. |
| **Personal usefulness before SaaS** | Every feature answers: *does this make my daily work better?* If the honest answer is "it helps future customers", defer it. |
| **Data quality over feature count** | A correct finding beats three more screens. Time spent on validation, plausibility checks and classification accuracy is the highest-return time in the build. |
| **Reliability over complexity** | Duplicate prevention, resumability and idempotency are non-negotiable. Clever optimisations are. |

## 11.2 Avoid

| Anti-pattern | Why it kills projects |
|---|---|
| **Building SaaS features early** | Multi-tenancy is already in the schema. Anything beyond that serves nobody and delays what does. |
| **Overengineering** | Abstractions with one implementation are speculation. Build the second implementation before extracting the interface — except at adapter boundaries, where the interface is the point. |
| **Unnecessary AI features** | Eighteen jobs exist. A nineteenth must justify itself with a tier and an evaluation suite, or it does not ship. |
| **Dashboards before the workflow works** | Analytics over an empty database is decoration. M10 is last for a reason. |
| **Horizontal building** | The most common failure mode. Vertical slices, always. |

## 11.3 Non-negotiables

Three things never get cut, regardless of schedule pressure:

| Non-negotiable | Reason |
|---|---|
| **Legal gate** | Guards the only risk that can end the business; far harder to retrofit |
| **Design evaluation gate** | Without it there is no automation, only faster mistakes |
| **Outcome recording** | Data not captured is permanently lost; it is the moat's raw material |

Plus four correctness guarantees:

| Guarantee | Consequence if broken |
|---|---|
| Baseline shown on every finding | Findings mislead |
| Evidence suppression thresholds enforced | Noise presented as insight |
| Duplicate prevention | The worst operational failure possible |
| Margin floor enforcement | Unintended loss-making |

## 11.4 Cut list — in order

If the schedule slips, cut from the top:

```
   1.  Vectorisation                    raster is publishable
   2.  Interaction-effect detection     second-order insight
   3.  Crowded-loser detection          a refinement of failure analysis
   4.  Run comparison over time         needs history to be useful anyway
   5.  Global search                    convenience
   6.  Report export                    convenience
   7.  Concept expansion                the base twenty suffice
   8.  Manual concept entry             narrow use case
   9.  Deep research depth              Standard delivers the value
   10. Seasonality profiling            useful but not load-bearing
```

## 11.5 Weekly discipline

| Practice | Purpose |
|---|---|
| One demo per milestone, to yourself | Forces end-to-end thinking |
| Run the failure-injection script before each milestone exit | Untested resilience is absent resilience |
| Review cost telemetry weekly | Cost drift compounds silently |
| Do not start a milestone before the previous one meets its criteria | Partial completion accumulates into a system that never works |

---

# 12 — Executive Summary

## 12.1 What should I build first?

**Milestone 0 — five days of throwaway analysis, no code.**

Collect ~150 listings across three niches, classify their visual attributes by hand, compute cohorts and lift, produce a findings list, and review it blind against four categories with planted false controls.

**Then M1 — the walking skeleton.** Two weeks establishing the orchestrator, the database, the provider fakes and the offline development environment. It demos as a progress bar and it is the most important infrastructure in the build. Every later milestone depends on it and inherits its weaknesses.

**Then follow the critical path.** Do not reorder it. In particular: analysis before generation, legal gate before artwork, artwork evaluation before commerce.

---

## 12.2 What should I ignore for now?

| Ignore | Until |
|---|---|
| Multi-user, teams, billing | V3 — the schema already supports it |
| Browser extension | V2 — four providers already work |
| Active weight recalibration | V2 — needs ~50 outcomes |
| Bulk publishing | V2 — and it works against the product thesis |
| Additional marketplaces | V3+ |
| Public API, mobile, white-label | V3+ |
| Dashboards | M10 — analytics over an empty database is decoration |
| Elegant abstractions | The second use case |

**And permanently ignore:** automatic publishing, volume listing tools, design template libraries, and any technique for evading detection when gathering data.

---

## 12.3 What creates the most value?

**Milestone 4 — the analysis engines.**

Everything before it is enabling work. Everything after it exploits what it produces. If M4 works, the product has a reason to exist even if nothing after it is built.

Within M4, the highest-value component is the **visual analysis feeding the success and failure engines** — converting aesthetics into statistics is the thing no competitor tool does and the thing a human cannot do by eye across four hundred listings.

**Second: Milestone 9 — publishing.** It removes 45 minutes of mechanical work per product, which is the most immediately felt improvement to daily life.

**Third, and the one that compounds: outcome recording in M10.** It produces no visible value on the day it ships and becomes the entire moat over eighteen months.

---

## 12.4 What is the biggest risk?

**Business risk — the analysis confirms rather than reveals.**

If the findings are things you already knew, the research layer adds nothing and the product collapses to publishing automation — useful, but far less valuable and far more replaceable.

*Mitigated by M0 before any code exists, and re-tested at the M4 checkpoint.*

**Technical risk — irreversible marketplace actions.** A duplicate listing, an accidental publish, or a product that sells but never manufactures. Every other technical failure is recoverable; these are externally visible and damage reputation.

*Mitigated by four-layer duplicate prevention, draft-only automation, server-side pre-flight, mandatory fulfilment linking, and chaos testing as a milestone gate.*

**Execution risk — building horizontally and losing momentum.** Months on a beautiful data model with no working pipeline, shelved at 70% complete having produced nothing.

*Mitigated by vertical slices, a demo per milestone, the M4 checkpoint, and a cut list that allows scope reduction without abandonment.*

---

## 12.5 What determines whether this becomes a valuable product?

**Four things, in order.**

### 1. Whether the analysis reveals something you did not know

Everything rests on this. It is testable in five days and is the entire purpose of Milestone 0. If the answer is no, reposition immediately rather than building for five months on a false premise.

### 2. Whether generated artwork is usable without significant rework

If each design needs forty minutes of manual fixing, the automation promise fails and the product becomes research-only. The six-check evaluation gate exists to make this measurable rather than a matter of impression.

### 3. Whether published products actually outperform your manual baseline

The honest test, and it takes 90 days after M9 to answer. If products chosen by the system do not outperform products chosen by intuition, the loop does not close and the learning phase will not rescue it.

### 4. Whether outcome data accumulates from day one

The moat is the dataset linking design attributes to realised sales. It is impossible to buy, compounds monotonically, and **cannot be reconstructed retrospectively**. Every month of publishing without recording is a month of the asset permanently lost.

---

## 12.6 The one-paragraph version

> Spend five days proving the analysis reveals non-obvious things before writing any code. Then build a solid skeleton in two weeks, get real market data flowing, and get the four analysis reports working — that is the halfway point and the point at which the product is worth using. Then add concepts, the legal gate, artwork with its evaluation gate, commerce and publishing, in that order, without reordering. Record outcomes from the first published listing. Ignore everything to do with multiple users until you have used it yourself, daily, for a month. **Plan for 25 weeks; the product is genuinely useful at week 13 and complete at week 27.**

---

## Document control

| | |
|---|---|
| **Phase** | 1D — Master Implementation Roadmap |
| **Scope** | V1 build sequence; V2 and V3 defined but not planned in detail |
| **Excludes** | Application code; detailed UI design |
| **Next** | Execution — beginning with Milestone 0 |
| **Status** | Ready for execution |
