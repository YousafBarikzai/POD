# POD Intelligence — Phase 1D

# Implementation Plan

**Document type:** Build Plan
**Phase:** 1D — *how the design becomes working software*
**Version:** 1.0
**Status:** For engineering review
**Prerequisites:** [Part 1A — Product Definition](PART-1-product-definition.md) · [Phase 1B — Technical Architecture](PHASE-1B-technical-architecture.md) · [Phase 1C — AI Systems](PHASE-1C-ai-systems-and-integrations.md)

> **Scope.** This document converts three phases of design into an executable build sequence for **Version 1 — the personal tool**. It covers what to build, in what order, why that order, how to know each stage is finished, and how to detect early that the plan is going wrong.
>
> V2 and V3 are out of scope here — their entry conditions are defined in [Phase 1C §16](PHASE-1C-ai-systems-and-integrations.md#16--version-roadmap) and neither begins until V1 is in daily use.

---

## Contents

| § | Section |
|---|---|
| 1 | [Build Philosophy](#1--build-philosophy) |
| 2 | [The Dependency Graph](#2--the-dependency-graph) |
| 3 | [Milestone Overview](#3--milestone-overview) |
| 4 | [Milestone 0 — Thesis Validation](#4--milestone-0--thesis-validation) |
| 5 | [Milestone 1 — Walking Skeleton](#5--milestone-1--walking-skeleton) |
| 6 | [Milestone 2 — Real Market Data](#6--milestone-2--real-market-data) |
| 7 | [Milestone 3 — Visual Analysis](#7--milestone-3--visual-analysis) |
| 8 | [Milestone 4 — The Analysis Engines](#8--milestone-4--the-analysis-engines) |
| 9 | [Milestone 5 — Concepts & Scoring](#9--milestone-5--concepts--scoring) |
| 10 | [Milestone 6 — Legal Gate](#10--milestone-6--legal-gate) |
| 11 | [Milestone 7 — Artwork Pipeline](#11--milestone-7--artwork-pipeline) |
| 12 | [Milestone 8 — Commerce](#12--milestone-8--commerce) |
| 13 | [Milestone 9 — Publishing](#13--milestone-9--publishing) |
| 14 | [Milestone 10 — Tracking & Hardening](#14--milestone-10--tracking--hardening) |
| 15 | [Local Development Setup](#15--local-development-setup) |
| 16 | [Testing Gates](#16--testing-gates) |
| 17 | [Estimates](#17--estimates) |
| 18 | [Risk Register & Early Warning Signals](#18--risk-register--early-warning-signals) |

---

# 1 — Build Philosophy

## 1.1 Five sequencing principles

### Principle 1 — Validate the thesis before building the machine

**The product rests on one unproven assumption:** that statistical analysis of competitor listings produces findings the operator did not already know and would act on.

If that assumption is false, every subsequent milestone is wasted work. It is testable in roughly a week with throwaway scripts and no infrastructure.

**Therefore Milestone 0 exists, and nothing else starts until it passes.**

### Principle 2 — Vertical slices, not horizontal layers

Do **not** build "all the database", then "all the API", then "all the UI". Build thin complete paths through every layer, then widen them.

| Horizontal (wrong) | Vertical (correct) |
|---|---|
| Nothing works until the end | Something works from week 3 |
| Integration problems surface last | Integration problems surface first |
| No feedback until complete | Feedback continuously |
| Estimation error compounds silently | Estimation error visible early |

### Principle 3 — Build the riskiest thing early, within each slice

Within a milestone, sequence by uncertainty rather than by convenience. The purpose of early work is to convert unknowns into knowns while there is still time to change the plan.

**Ranked uncertainty in this build:**

| Rank | Unknown | Resolved by |
|---|---|---|
| 1 | Does the analysis produce useful findings? | Milestone 0 |
| 2 | Can we obtain adequate market data? | Milestone 2 |
| 3 | Is visual classification accurate enough? | Milestone 3 |
| 4 | Does artwork generation produce print-ready output reliably? | Milestone 7 |
| 5 | Does publishing work without duplicates? | Milestone 9 |

### Principle 4 — Fakes before integrations

Every external provider gets a fixture-backed fake **before** the real adapter. This means:

- The full pipeline runs offline, free, deterministically, from Milestone 1.
- A provider outage never blocks development.
- Tests are fast and never flaky.
- The real adapter is a swap, not a rewrite.

### Principle 5 — The three non-deferrables stay in scope

From Phase 1C §16.5, these do not move regardless of schedule pressure:

| Non-deferrable | Why |
|---|---|
| **Legal gate** | Guards the only risk that can end the business; far harder to retrofit into a system already generating artwork |
| **Design evaluation gate** | Without it there is no automation, only faster mistakes |
| **Learning-loop recording** | Outcome data cannot be recovered retrospectively — a month not recorded is permanently lost |

**If the schedule slips, cut breadth, not these.**

---

## 1.2 What "done" means

A milestone is done when **all** of the following hold. Partial completion is not completion.

- [ ] Every listed capability works against real or fixture data as specified
- [ ] Tests at the required level pass
- [ ] The demo scenario runs end to end without manual intervention
- [ ] Observability exists for anything new — logs, metrics, cost attribution
- [ ] The exit criteria are met and verified, not asserted

---

# 2 — The Dependency Graph

## 2.1 Hard dependencies

```
   M0  THESIS VALIDATION
       (throwaway — no production code)
        │
        ▼  proves the product is worth building
   M1  WALKING SKELETON
       monorepo · database · auth · orchestrator · queue · UI shell
       ALL providers faked
        │
        ├──────────────────────────────┐
        ▼                              ▼
   M2  REAL MARKET DATA           M3  VISUAL ANALYSIS
       provider layer                  colour measurement
       Etsy API · CSV · manual         classification · embeddings
        │                              │
        └──────────────┬───────────────┘
                       ▼
   M4  ANALYSIS ENGINES         ★ THE PRODUCT'S CORE VALUE ★
       opportunity · success · failure · gap
                       │
                       ▼
   M5  CONCEPTS & SCORING
       generation · deduplication · six scores
                       │
                       ▼
   M6  LEGAL GATE               ★ BLOCKS M7 BY DESIGN ★
                       │
                       ▼
   M7  ARTWORK PIPELINE
       brief · generation · processing · evaluation gate
                       │
                       ▼
   M8  COMMERCE
       product recommendation · pricing · SEO
                       │
                       ▼
   M9  PUBLISHING
       fulfilment · marketplace draft · review · publish
                       │
                       ▼
   M10 TRACKING & HARDENING
       performance sync · analytics · outcome recording
```

## 2.2 What can run in parallel

With more than one engineer:

| Can parallelise | Notes |
|---|---|
| **M2 and M3** | Independent inputs to M4. The largest parallelisation opportunity. |
| M4's four engines | Share statistical primitives; build those first, then split |
| M6 and M7 groundwork | Legal adapters and artwork adapters are independent |
| M8's three areas | Product, pricing and SEO are largely independent |
| UI and backend within any milestone | Given contracts are agreed first |

| Cannot parallelise | Reason |
|---|---|
| M1 before anything | Everything depends on the skeleton |
| M4 before M2 and M3 | Nothing to analyse |
| M7 before M6 | The gate must exist before generation, or it will be retrofitted badly |
| M9 before M8 | Nothing to publish |

## 2.3 The critical path

```
   M0 → M1 → (M2 ∥ M3) → M4 → M5 → M6 → M7 → M8 → M9 → M10
```

**M4 is the widest point of risk.** It is where the product's value is either demonstrated or not. Everything before it is enabling work; everything after it is exploitation. Budget accordingly, and do not compress it.

---

# 3 — Milestone Overview

| M | Name | Duration | Delivers | Demo |
|---|---|---|---|---|
| **0** | Thesis validation | 1 week | Evidence the product is worth building | Findings from a real niche the operator did not already know |
| **1** | Walking skeleton | 2 weeks | Runnable system, all providers faked | Start a fake run, watch it stream, kill a worker, watch it resume |
| **2** | Real market data | 2 weeks | Provider layer with real data flowing | Real competitor data collected from a real niche |
| **3** | Visual analysis | 1.5 weeks | Structured visual attributes | 400 listings classified with measured accuracy |
| **4** | Analysis engines | 3 weeks | **The core value** | All four reports on a real niche |
| **5** | Concepts & scoring | 2 weeks | 20 grounded concepts, six scores | Concept board with evidence citations resolving |
| **6** | Legal gate | 1.5 weeks | Screening that cannot be bypassed | Blocked concept cannot generate artwork, verified at service layer |
| **7** | Artwork pipeline | 2.5 weeks | Print-ready artwork | Four variants through the six-check gate |
| **8** | Commerce | 2 weeks | Product, pricing, listings | Ranked configurations with real margins, 10 SEO variations |
| **9** | Publishing | 2 weeks | Live listings | Product published end to end, zero duplicates under chaos |
| **10** | Tracking & hardening | 2 weeks | Outcomes flowing, system solid | Performance data flowing, full lineage traceable |
| | **Total** | **21.5 weeks** | | |

**Elapsed calendar time will exceed this.** See §17 for the estimate model and buffer recommendation.

---

# 4 — Milestone 0 — Thesis Validation

**Duration:** 1 week · **Produces:** throwaway analysis scripts and a written verdict · **Produces no production code**

## 4.1 Why this exists

The product's central claim is that statistical analysis of competitor listings reveals **non-obvious, actionable patterns**. Three ways that claim could fail:

| Failure mode | Consequence |
|---|---|
| Findings are obvious — the operator already knew them | The research layer adds no value; the product is a publishing tool |
| Findings are noise — patterns that do not replicate | The product is actively harmful, giving confident wrong advice |
| Findings are real but not actionable | Interesting, but does not change what gets made |

**Each is fatal, and each is cheap to test now and expensive to discover at Milestone 4.**

## 4.2 Method

| Step | Detail |
|---|---|
| 1 | Pick **three niches** the operator knows well — one they believe they understand, one they are unsure about, one they have never worked in |
| 2 | Gather data by whatever means is fastest — manual export, spreadsheet, hand collection. **No infrastructure.** ~100–200 listings per niche is sufficient |
| 3 | Classify visual attributes by hand or with ad-hoc model calls. Accuracy matters more than automation here |
| 4 | Compute cohorts, prevalence, baselines, lift and significance in a spreadsheet or a script |
| 5 | Produce a findings list for each niche |
| 6 | **The operator reviews each finding blind**, before seeing the statistics, and marks it: *already knew · suspected · surprising · disagree* |

## 4.3 Exit criteria

| Criterion | Threshold |
|---|---|
| **Non-obvious findings** | At least **30%** of significant findings marked *surprising* or *suspected but unconfirmed* |
| **Actionability** | The operator can name at least **three specific design decisions** they would make differently |
| **Cross-niche consistency** | The method produces coherent findings in all three niches, including the unfamiliar one |
| **No absurdities** | No finding the operator judges obviously wrong |
| **Sample viability** | Sufficient listings pass evidence thresholds at realistic collection volumes |

## 4.4 If it fails

**Stop and reconsider before building.** This is the point of the milestone.

| Outcome | Response |
|---|---|
| Findings obvious | Reposition: lead with gap detection and publishing automation, treat success analysis as supporting |
| Findings noisy | Raise evidence thresholds, increase collection volume, re-test. If still noisy, the statistical layer needs rethinking. |
| Findings unactionable | Rework the synthesis output — the finding list may be right but the presentation wrong |
| Data unobtainable at volume | Re-scope to the depths that are achievable |

**A week spent here can save five months.** It is the highest-return week in the entire plan.

---

# 5 — Milestone 1 — Walking Skeleton

**Duration:** 2 weeks · **Delivers:** a runnable system with every architectural seam exercised and every provider faked

## 5.1 Goal

**Every hard architectural problem is solved and demonstrated before any feature is built.** No feature work should discover that the orchestrator does not resume, or that progress streaming does not reconnect.

## 5.2 Sprint 1.1 — Foundations (week 1)

| Task | Detail |
|---|---|
| Monorepo | Workspaces, build orchestration, shared type and lint configuration |
| **Dependency boundary enforcement** | The rule file from Phase 1B §K.10, failing the build on violation. **Build this now** — retrofitting boundaries after they have eroded is not feasible. |
| Contracts package | Shared types, enums, validation schemas |
| Configuration | Environment schema validated at boot; process refuses to start on invalid config |
| Database schema | The full schema from Phase 1B §C — all tables, including partitioning declarations and isolation policies |
| Repository layer | The base pattern plus repositories for run, niche, shop, listing |
| **Seed fixture** | A complete realistic workspace: finished run, 10 shops, ~400 listings with visual attributes, all four reports, 20 concepts, 4 artworks, 2 published listings with performance history |
| Local stack | Containers for database, cache, object storage, mail, tracing |
| Build pipeline | Lint, typecheck, test, build |

**The seed fixture is not optional and not a nice-to-have.** Without it, every screen is built against an empty state, layout and performance problems surface only in production, and later milestones cannot be worked on without manually producing earlier-stage data.

## 5.3 Sprint 1.2 — Platform (week 2)

| Task | Detail |
|---|---|
| Authentication | Password hashing, second factor, sessions, lockout, recovery codes |
| Queue abstraction | Wrapper over the queue library, job registry, dead-letter handling |
| **Orchestrator** | Step graph definition, step runner, dependency resolution, budget guard, heartbeats, reaper, resume |
| Observability | Structured logging with bound context, tracing, metrics, cost telemetry |
| **Provider fakes** | Fixture-backed fakes for **every** provider — market data, AI, image generation, marketplace, fulfilment, registries — with configurable latency and failure injection |
| UI shell | Layout, navigation, theming, design tokens, core components |
| API wiring | Procedure classes, middleware chain, error model, idempotency |
| Progress streaming | Server-sent events with publish-subscribe fan-out, reconnection, polling fallback |
| **Demo pipeline** | Three fake steps that run, stream progress, incur fake cost, fail, retry, resume |

## 5.4 Exit criteria

- [ ] A run starts, streams progress, and completes
- [ ] **A worker killed mid-run resumes and produces identical output**
- [ ] Cancellation stops work within seconds and preserves completed steps
- [ ] Budget exhaustion pauses the run rather than overspending
- [ ] Progress stream reconnects after a dropped connection without losing events
- [ ] The full pipeline runs **offline with zero external calls and zero cost**
- [ ] Seed data renders realistically on every implemented screen
- [ ] Dependency boundary violations fail the build
- [ ] The application is deployed to staging and reachable

## 5.5 The trap to avoid

**Under-investing in the orchestrator because it produces no visible feature.**

It is two weeks of work that demos as "a progress bar". Every subsequent milestone depends on it. A weak orchestrator means every later milestone carries hidden reliability debt that surfaces as intermittent failures nobody can reproduce.

**Resist the pull to start on reports.**

---

# 6 — Milestone 2 — Real Market Data

**Duration:** 2 weeks · **Delivers:** the provider layer with real data flowing

## 6.1 Sprint 2.1 — The provider layer (week 1)

| Task | Detail |
|---|---|
| **Normalised contract** | The shop, listing and keyword record shapes from Phase 1C §3.3, with the provenance envelope |
| Provider interface | Capability declaration, probe, fetch |
| Chain resolution | Probe available providers, resolve order, handle absence |
| **Field-level merge** | Precedence by measured-over-estimated, then confidence, then freshness, with per-field source recorded |
| **Etsy API provider** | Authorisation flow, shop and listing reads, taxonomy, rate limiter with reserved allocation |
| **CSV import provider** | Format detection, column mapping with memory, streamed parsing, validation including plausibility and cross-field checks, import reporting |
| **Manual entry provider** | Tabular entry supporting spreadsheet paste, confidence declaration, editability |

**Build all three green providers in the first week.** This is what makes the layer genuinely modular rather than EverBee-with-fallbacks, and it is the refinement that removed single-vendor dependency.

## 6.2 Sprint 2.2 — Collection (week 2)

| Task | Detail |
|---|---|
| Shop discovery | Candidate assembly from multiple signals, deduplication |
| **Selection scoring** | The formula from Phase 1C §4.2 including the age multiplier and the 20/10/5 fallback ladder |
| Qualification gates | With exclusion reasons recorded |
| Listing collection | Relevance filtering, per-shop and per-run caps, **truncation reporting** |
| Snapshot persistence | Immutable dated observations with provenance |
| Asset capture | Image download, content hashing, deduplication |
| EverBee export provider | Format detection for known export layouts |
| Competitor report UI | Shop table, listing table, distributions, detail drawer |
| **Capability degradation** | Banners, confidence downgrades, method disclosure |

## 6.3 Exit criteria

- [ ] Real competitor data collected from a real niche using **Etsy API alone**
- [ ] The same run works with a CSV import added, with merged provenance visible
- [ ] Manual entry produces records that participate correctly in the merge
- [ ] Disabling each provider in turn degrades gracefully with the correct banner
- [ ] The fallback ladder triggers correctly and is visible in the interface
- [ ] Truncation is reported when caps are hit
- [ ] Every displayed value shows its source and whether it is estimated
- [ ] A 100,000-row import completes within the performance budget

## 6.4 De-risking note

**Test the Etsy-API-only path first and hardest.** It is the path that must always work. If development proceeds with rich imported data throughout, the degraded path will be under-tested and will fail on first real use.

---

# 7 — Milestone 3 — Visual Analysis

**Duration:** 1.5 weeks · **Delivers:** structured visual attributes with measured accuracy

## 7.1 Tasks

| Task | Detail |
|---|---|
| **Colour measurement** | Perceptual-space clustering, palette family mapping, dominant colour, contrast profile, colour count — **deterministic, no model** |
| Vision classification | Typography, layout, mockup style, illustration style, complexity, subject, humour, garment colour — constrained vocabularies |
| **Content-hash caching** | Keyed by image content and analyser version. Build this **with** the classifier, not after — it changes the call pattern. |
| Batching | Multiple images per call |
| Embeddings | Visual embeddings for similarity and originality |
| **Golden set** | 200 hand-labelled thumbnails for accuracy measurement |
| Evaluation harness | Per-class accuracy measurement against the golden set |
| Failure handling | Excluded listings retained, **exclusion count reported** |

## 7.2 Exit criteria

- [ ] Classification accuracy meets the thresholds on the golden set
- [ ] Colour extraction is **reproducible** — identical input yields identical output every time
- [ ] Cache hit rate exceeds the target on a repeat run in the same niche
- [ ] A batch of 400 listings completes within the latency and cost budgets
- [ ] Out-of-vocabulary model output is **rejected**, not coerced
- [ ] The exclusion count is displayed when analysis fails for some images

## 7.3 The accuracy question

**If classification accuracy is below threshold, stop and fix it before Milestone 4.**

Every finding in the success and failure engines depends on these attributes. Poor classification produces confident findings about attributes that were never really measured — which is worse than no findings, because it will be acted upon.

**Remediation order:** improve the prompt and its examples → increase the golden set and re-measure → promote the model tier → narrow the vocabulary if a class is genuinely ambiguous.

---

# 8 — Milestone 4 — The Analysis Engines

**Duration:** 3 weeks · **Delivers: the product's core value**

**This is the most important milestone. Do not compress it.**

## 8.1 Sprint 4.1 — Statistical primitives (week 1)

Build the pure functions first. Everything else in this milestone depends on them.

| Task | Detail |
|---|---|
| Cohort assignment | Age-normalised performance with the age cap, exposure filtering |
| Contingency analysis | Cohort support, baseline support, lift, significance testing |
| Numeric analysis | Effect size, optimal band search |
| Confidence assignment | From sample size and significance |
| Suppression | Below-threshold findings flagged and excluded from ranking |
| **Cross-shop representation check** | Prevents one shop's house style becoming a "market finding" |
| Factor weighting | The lift-confidence-sample formula |
| Correlation and interaction | Rank correlation, multicollinearity flagging, pairwise interaction |
| **Property tests** | Monotonicity, bounds, invariance under irrelevant permutation |

**These are pure functions with no input or output.** They should be exhaustively tested, run in milliseconds, and require no network. This is the package that must be most correct in the entire system.

## 8.2 Sprint 4.2 — Success and failure engines (week 2)

| Task | Detail |
|---|---|
| Success engine | Cohorting, per-attribute analysis, weighting, statement rendering from computed values |
| **Synthesis card** | The median winning listing specification |
| Style resolution | Automatic style selection with evidence |
| Failure engine | Same treatment inverted, with the 90-day exposure requirement |
| **Causality labelling** | Causally plausible versus correlation-only |
| Ambiguity handling | Net lift, ambiguous marking, weight reduction |
| Crowded-loser detection | Common-and-failing combinations |
| Do/Avoid sheet | The printable comparison |
| Report UI | Ranked findings with **baseline always displayed**, evidence drill-through |

## 8.3 Sprint 4.3 — Opportunity and gap engines (week 3)

| Task | Detail |
|---|---|
| Sub-niche discovery | Model expansion merged with co-occurrence and taxonomy signals |
| Opportunity sub-scores | Demand, competition, trend, profitability, seasonality |
| Verdict banding | With executive summary |
| Seasonality profiling | Twelve-month index, peaks, troughs, strength |
| Coverage matrix | Sub-niche × angle × style with supply and demand |
| **Demand floor** | Exclusion with **reasons retained and displayed** |
| Gap scoring | The weighted formula with feasibility |
| Caution flags | Trademark-heavy, seasonally dead, unprintable |
| Gap UI | Bubble map with the floor drawn, ranked list, coverage heat map |

## 8.4 Exit criteria

- [ ] All four reports produced from a real niche within the latency and cost budgets
- [ ] **Every finding displays cohort support, baseline, lift, sample size and confidence**
- [ ] Findings below thresholds appear in a separate section, never in the ranked list
- [ ] A finding from a single shop is suppressed by the representation check
- [ ] The demand floor excludes low-demand cells, and excluded cells remain visible with their reason
- [ ] Evidence drill-through resolves to the specific listings behind every finding
- [ ] **Scores are reproducible** — re-running against stored features yields identical values
- [ ] The synthesis card is usable as a design brief without further interpretation
- [ ] Automatic style selection produces a defensible winner with evidence

## 8.5 The checkpoint that matters

**At the end of Milestone 4, run the Milestone 0 test again — with the real system, on a fresh niche.**

Does it produce findings the operator did not already know and would act on?

| Result | Action |
|---|---|
| Yes | **Proceed.** The thesis holds at production quality. |
| No | **Stop.** Something regressed between the spike and the implementation. Diagnose before building anything on top. |

This checkpoint is the difference between discovering a fundamental problem now and discovering it after four more months of work.

---

# 9 — Milestone 5 — Concepts & Scoring

**Duration:** 2 weeks

## 9.1 Sprint 5.1 — Generation (week 1)

| Task | Detail |
|---|---|
| **Grounding assembly** | The compact statistical context. **Verify it contains no competitor identifiers, titles, descriptions or image references** — this is the originality guarantee, and it must be tested, not assumed. |
| Success-derived generation | 10 concepts with citations |
| Gap-derived generation | 10 concepts with citations, run in parallel |
| Output contracts | Validation, repair, stricter retry |
| Embedding and deduplication | Within-set rejection, history flagging, quota refill |
| **Groundedness evaluation** | Every cited finding must exist and match its stored statistic — gate at 100% |
| Concept board UI | Cards, filters, sorting, selection, drawer with citations |

## 9.2 Sprint 5.2 — Scoring (week 2)

| Task | Detail |
|---|---|
| Category weights | The weighting system from Phase 1C §5.4 as a versioned configuration |
| Market Fit | Category alignment, opportunity positioning, anti-finding penalties |
| Originality | Embedding distance with self-similarity discount |
| Competition | Comparable density with concentration and quality-ceiling modifiers |
| Visual Quality | Provisional at concept stage, with renormalised weights |
| POD Suitability | Provisional at concept stage |
| **Estimated Potential** | The weighted composite **multiplied** by the suitability factor |
| Contribution vectors | Itemised signed contributions, persisted |
| **Reasoning rendering** | Generated from contributions, never free-form |
| Confidence levels | From evidence volume, quality, model maturity, data completeness |
| Regeneration and expansion | Single, bulk, steered, axis expansion, manual entry |
| **Language compliance** | Every surface uses estimate language per Phase 1C §10.0 |

## 9.3 Exit criteria

- [ ] Twenty genuinely distinct concepts, verified by embedding distance
- [ ] Every citation resolves and matches its stored statistic
- [ ] **No competitor identifier appears anywhere in the generation path** — verified by inspection and test
- [ ] Scores are reproducible for identical inputs
- [ ] Reasoning matches the score exactly, because both derive from the same computation
- [ ] Confidence levels reflect actual evidence; below threshold, no score is shown
- [ ] Concept regeneration preserves unaffected concepts
- [ ] **No interface surface uses prediction or guarantee language**

---

# 10 — Milestone 6 — Legal Gate

**Duration:** 1.5 weeks · **Non-deferrable**

## 10.1 Tasks

| Task | Detail |
|---|---|
| Entity extraction | Brands, characters, people, teams, slogans, media titles |
| Normalisation | Case, punctuation, spacing, common obfuscations |
| Internal blocklist | High-risk terms |
| **Registry adapters** | Multiple jurisdictions, relevant goods classes, exact/fuzzy/phonetic matching |
| Registry caching | By normalised term |
| **Registry failure handling** | Reduced coverage recorded and **explicitly surfaced — never a silent clean result** |
| Marketplace policy terms | Prohibited and restricted lists |
| Copyright assessment | Derivative indicators, likeness, quoted material |
| **Risk rule table** | Deterministic, versioned, table-driven |
| Safer alternatives | Generation and automatic re-screening |
| **Service-layer gate** | Artwork generation asserts screening state **where the external call is made** |
| Override flow | Step-up authentication, typed confirmation, written justification, audit |
| Burned-terms list | Permanent workspace blocking |
| **Adversarial evaluation** | Known marks, characters, slogans, quoted material |

## 10.2 Exit criteria

- [ ] **Attempting artwork generation for a blocked concept fails at the service layer and makes zero provider calls** — verified by direct call, bypassing the interface entirely
- [ ] Recall on the adversarial set meets the release gate
- [ ] The rule table produces identical risk levels for identical inputs
- [ ] Registry failure surfaces reduced coverage, never a clean pass
- [ ] High-risk override requires re-authentication, typed confirmation and justification, all recorded
- [ ] **Blocked concepts have no override path** — the endpoint rejects unconditionally
- [ ] Safer alternatives generate and re-screen automatically

## 10.3 Why the bypass test is the critical one

The gate must hold against a client that does not cooperate. Test it by calling the artwork service **directly**, with a blocked concept, bypassing every interface control.

If that call succeeds, the gate is decorative.

---

# 11 — Milestone 7 — Artwork Pipeline

**Duration:** 2.5 weeks

## 11.1 Sprint 7.1 — Generation (week 1)

| Task | Detail |
|---|---|
| **Deterministic preparation** | Palette derived from the winning family, dimensions computed, constraints assembled |
| Brief authoring | Model-authored within deterministic bounds |
| Brief editing | Operator review with versioning |
| **Prompt compilation** | In the adapter, not in prompt files — provider-specific tuning in one place |
| Style templates | One per design style |
| Ideogram adapter | With provider-side rewriting **disabled** |
| Variant generation | Independent calls, deterministic seeds |
| Ingestion | Download, hash, deduplicate, store originals immutably |

## 11.2 Sprint 7.2 — Processing (week 1.5)

| Task | Detail |
|---|---|
| Background removal | Alpha extraction, edge refinement, halo decontamination, island removal |
| Auto-crop | Content bounds with padding |
| Upscale | To print resolution with the excessive-factor guard |
| Renditions | Print, web, thumbnail, transparency proof |
| Vectorisation | Where suitable, with quality verification |

## 11.3 Sprint 7.3 — The evaluation gate (week 2.5)

**Build all six checks. This is the automation.**

| Check | Detail |
|---|---|
| 1 · Print quality | Every criterion with measured value, threshold and **specific remedy** |
| 2 · **Text accuracy** | Transcribe and compare **exactly** against the brief. Zero tolerance. Also verify no text where none was specified. |
| 3 · Visual quality | The weighted composite with thumbnail legibility highest |
| 4 · POD suitability | Garment compatibility, ink coverage, colour practicality, format fit |
| 5 · Similarity and copyright | Embedding distance plus vision safety review |
| 6 · **Market attribute match** | Re-analyse the render with the **same** classifier used on competitors, compare against brief and findings |
| Remediation | The decision tree routing each failure to the cheapest fix |
| **Diagnostic regeneration** | Brief amended from the dominant failure, then regenerate |
| Studio UI | Canvas, variants, quality panel, originality panel, tools, lineage |

## 11.4 Exit criteria

- [ ] Four variants generated within the cost budget including retries
- [ ] **Artwork failing any blocking check cannot be attached to a product**
- [ ] **A misspelled render is caught by the text accuracy check** — verified with a deliberately induced case
- [ ] Attribute drift is detected and reflected in the artwork-stage Market Fit score
- [ ] Each failure names its specific remedy, never a generic message
- [ ] Diagnostic regeneration amends the brief rather than blindly retrying
- [ ] Automatic retries are capped; beyond the cap the operator sees a diagnosis
- [ ] Originality warnings require acknowledgement before acceptance

## 11.5 The check most likely to be skipped

**Text accuracy.** It appears fussy until the first misspelled shirt reaches a customer.

Test it deliberately: generate artwork with text, corrupt the render, and verify the check catches it. Do not assume the model transcribes reliably — verify.

---

# 12 — Milestone 8 — Commerce

**Duration:** 2 weeks

## 12.1 Sprint 8.1 — Product and pricing (week 1)

| Task | Detail |
|---|---|
| Catalogue synchronisation | Structures weekly, **costs daily** |
| **Cost drift monitoring** | Threshold alerting, automatic repricing of affected drafts |
| Product recommendation | The weighted score with profitability highest |
| Colour recommendation | Empirical, **validated against artwork ink** |
| **Fee stack** | Every component, with advertising fees modelled as charged |
| Price solving | Target-margin and fixed-price modes |
| **Margin floor** | Blocking, with the required price stated |
| Competitor price context | Distribution with the chosen price marked |
| Pricing UI | Itemised waterfall, both advertising cases shown |

## 12.2 Sprint 8.2 — Listings (week 2)

| Task | Detail |
|---|---|
| Keyword pool | Frequency, relevance against baseline, **sales weighting**, top-decile usage, competition index |
| Generation | Ten variations along declared positioning axes |
| **Hard validation** | Marketplace constraints, one repair, then regenerate |
| Quality scoring | Coverage, front-loading, diversity, readability, **competitive fit targeting moderate not minimum** |
| Text screening | Restricted and trademark terms |
| Listing UI | Editor with live counters, keyword evidence drawer, preview |

## 12.3 Exit criteria

- [ ] Ranked configurations with real costs and correct margins
- [ ] **Both advertising cases displayed; the floor evaluated against the pessimistic one**
- [ ] Below-floor configurations blocked with the required price stated
- [ ] Colour recommendations flag artwork incompatibility
- [ ] A cost change beyond threshold triggers repricing and an alert
- [ ] Ten demonstrably distinct variations, each labelled with its axis
- [ ] Every keyword shows its evidence
- [ ] Constraint validation passes at 100% after repair
- [ ] No path can persist a listing the marketplace would reject

---

# 13 — Milestone 9 — Publishing

**Duration:** 2 weeks

## 13.1 Sprint 9.1 — Fulfilment (week 1)

| Task | Detail |
|---|---|
| Artwork upload | Content-hash deduplication |
| Placement | Computed per product type, operator-adjustable, persisted |
| Product creation | Idempotent, with reconciliation on ambiguous timeout |
| Mockup retrieval | Asynchronous with backoff, **never blocking a request** |
| Mockup selection | Filtered to a preferred set, ordered by niche performance |
| Error mapping | Provider errors to specific remedies |

## 13.2 Sprint 9.2 — Marketplace and approval (week 2)

| Task | Detail |
|---|---|
| Draft creation | **Always draft state. No path creates an active listing.** |
| Image upload | Ordered, per-image keys, targeted retry |
| Inventory | Full replacement, stock-keeping identifiers mapping to fulfilment |
| **Marketplace linking** | Mandatory, tracked, blocking check |
| Review page | Everything on one screen |
| **Checklist** | Hard versus soft, with jump-to-fix |
| **Server-side pre-flight** | Re-evaluate every hard check from current data |
| Publish | Explicit confirmation, audit record, sync scheduled |
| **Duplicate prevention** | All four layers |
| Targeted retry | Per-operation repair |

## 13.3 Exit criteria

- [ ] A product publishes end to end to a live listing
- [ ] **Zero duplicates under chaos testing** — worker killed mid-publish, timeouts induced
- [ ] Reconciliation adopts an existing listing rather than creating a second
- [ ] Publishing is refused server-side when any hard check fails, **even when the request claims otherwise**
- [ ] Partial failure repairs by targeted retry without recreating anything
- [ ] Authorisation loss pauses operations resumably rather than failing them
- [ ] Missing marketplace link **blocks publishing**
- [ ] Total operator attention per product is under the target

## 13.4 The test that must not be skipped

**Chaos testing for duplicates.** Kill the worker between the external call and the local record. Induce a timeout after a successful creation. Replay a request. Run the same publish twice concurrently.

**Zero duplicates, every time.** A duplicate listing on a live shop is the most operationally damaging failure this system can cause.

---

# 14 — Milestone 10 — Tracking & Hardening

**Duration:** 2 weeks

## 14.1 Sprint 10.1 — Tracking (week 1)

| Task | Detail |
|---|---|
| Performance sync | Daily, more frequent in the first week |
| Signals captured | **Views, clicks, favourites, orders, revenue, state** |
| Immutable snapshots | With deltas computed at write |
| Derived metrics | Conversion, views per day, days to first sale |
| Percentile computation | Nightly, age-adjusted |
| Drift detection | Surfaced, never silently overwritten |
| **Outcome recording** | Feature vectors built from day one — **non-deferrable** |
| Profit calculation | Real costs against real orders |
| Lineage | Full chain, every link navigable |

## 14.2 Sprint 10.2 — Analytics and hardening (week 2)

| Task | Detail |
|---|---|
| Portfolio dashboard | Revenue and orders over time with publication markers |
| Per-niche comparison | |
| **Concept-origin comparison** | Success-derived versus gap-derived |
| Accuracy view | Estimated against realised, calibration curve |
| Cost dashboard | Spend by activity and provider against budget |
| **Learning readiness** | Outcome count against threshold, stated plainly |
| Notifications | First sale, deactivation, zero views, budget thresholds |
| Global search | |
| Export | Reports and data |
| Backup verification | Restore drill |
| Runbooks | One per alert |
| Performance and load testing | |
| Security review | |

## 14.3 Exit criteria

- [ ] Performance data flowing daily for published listings
- [ ] **Outcome records building correctly** — verified by inspection, since this data cannot be recovered later
- [ ] Any listing traces fully back to its research run
- [ ] The accuracy view populates and shows estimated against realised
- [ ] Learning readiness shows the outcome count and the threshold **with a specific number**
- [ ] Cost tracking accurate against provider invoices
- [ ] Restore drill completed and recorded
- [ ] Every alert has a runbook
- [ ] Latency and cost budgets met under load

## 14.4 V1 acceptance

The full V1 exit criteria from Phase 1C §16.2, plus the one that actually decides it:

> **Does the creator prefer this to their previous process, and use it daily?**

If not, V1 is not finished — regardless of how many criteria are ticked.

---

# 15 — Local Development Setup

## 15.1 The requirement

**One command from a fresh clone to a working system with realistic data.** Anything more is friction that compounds across every day of the build.

```
   install dependencies
   start the local stack        database · cache · object storage · mail · tracing
   apply schema and seed        a complete realistic workspace
   run all processes            web · worker · scheduler, with hot reload
```

## 15.2 Provider mocking

**With mocking enabled, every provider is fixture-backed.** The full pipeline runs offline, free, in seconds.

| Capability | Purpose |
|---|---|
| Configurable latency | Test progress streaming and timeout handling realistically |
| Configurable failure rate | Exercise retry and circuit-breaker paths |
| Deterministic responses | Reproducible tests, reproducible bugs |
| Recorded real responses | Fixtures captured from real calls, so shapes are accurate |

**Mocking is rejected when the environment is production.** That combination prevents start-up.

## 15.3 The seed fixture

| Contains | Why |
|---|---|
| A completed run with 10 shops and ~400 listings | Reports have realistic content and realistic volume |
| Full visual attributes on those listings | Analysis engines have real input |
| All four reports | Report screens work from minute one |
| 20 concepts with scores and citations | Concept board is populated |
| 4 artworks with renditions and quality results | Artwork studio is populated |
| 2 product drafts at different stages | Commerce screens are populated |
| 2 published listings with 90 days of performance | Analytics and lineage are populated |

**Regenerated, not hand-maintained.** A script produces it, so it stays current as the schema evolves.

## 15.4 Failure injection

A script that exercises resilience paths on demand: provider outages, slow responses, rate limiting, worker termination mid-step, ambiguous timeouts after successful external calls.

**Run it before every milestone exit.** Resilience that is not tested is resilience that does not exist.

---

# 16 — Testing Gates

## 16.1 What runs when

| Gate | Runs | Blocks |
|---|---|---|
| Lint, typecheck | Every change | Merge |
| **Dependency boundaries** | Every change | Merge |
| Unit and property tests | Every change | Merge |
| Domain coverage threshold | Every change | Merge |
| Adapter tests against fixtures | Every change | Merge |
| **AI evaluation fast suite** | Every prompt change | Merge |
| Security scanning | Every change | Merge |
| End-to-end journeys | Before merge | Merge |
| **Isolation suite** | Every change | Merge |
| AI evaluation full suite | Nightly | Alert |
| Load and soak | Nightly | Alert |
| **Chaos** | Weekly and before each milestone exit | Milestone exit |
| Restore drill | Monthly | Recorded |

## 16.2 Per-milestone test requirements

| Milestone | Specific requirement |
|---|---|
| M1 | Chaos: worker killed mid-run produces identical output |
| M2 | Each provider disabled in turn; degradation correct |
| M3 | Golden-set accuracy meets thresholds |
| M4 | **Reproducibility: identical inputs yield identical scores** |
| M5 | Groundedness at 100%; no competitor identifier in the generation path |
| M6 | **Service-layer bypass test**; adversarial recall gate |
| M7 | Induced misspelling caught; blocking checks prevent attachment |
| M8 | Constraint validation at 100%; margin floor blocks |
| M9 | **Chaos: zero duplicates under all induced failures** |
| M10 | Outcome records verified by inspection |

## 16.3 The three tests that matter most

| Test | Why |
|---|---|
| **Reproducibility (M4)** | If scores are not reproducible, the learning loop cannot work and no score can be explained |
| **Legal bypass (M6)** | If the gate can be bypassed, it is decorative and the business risk is unmitigated |
| **Duplicate prevention (M9)** | A duplicate on a live shop is the worst operational failure the system can cause |

---

# 17 — Estimates

## 17.1 By milestone

| M | Name | Weeks | Person-days |
|---|---|---|---|
| 0 | Thesis validation | 1 | 5 |
| 1 | Walking skeleton | 2 | 10 |
| 2 | Real market data | 2 | 10 |
| 3 | Visual analysis | 1.5 | 7 |
| 4 | **Analysis engines** | **3** | **15** |
| 5 | Concepts & scoring | 2 | 10 |
| 6 | Legal gate | 1.5 | 7 |
| 7 | Artwork pipeline | 2.5 | 12 |
| 8 | Commerce | 2 | 10 |
| 9 | Publishing | 2 | 10 |
| 10 | Tracking & hardening | 2 | 10 |
| | **Total** | **21.5** | **106** |

## 17.2 By workstream

| Workstream | Person-days |
|---|---|
| Infrastructure, pipeline, environment | 12 |
| Database, repositories, migrations | 8 |
| Authentication and security | 6 |
| Orchestrator, queue, budget | 10 |
| Adapters — 8 providers with fixtures | 16 |
| **Pure domain logic — scoring, statistics, pricing** | **16** |
| Engines — 13 | 22 |
| Prompts and evaluation suites | 10 |
| User interface — ~24 routes | 26 |
| Testing across all levels | 14 |
| Observability and runbooks | 6 |
| **Total** | **146** |

## 17.3 Reconciling the two figures

The milestone view assumes **some parallelisation** — typically the user interface for a milestone being built alongside its backend once contracts are agreed. The workstream view is total effort.

| Team | Elapsed calendar time |
|---|---|
| **One engineer, sequential** | ~29 weeks |
| **One engineer with substantial AI-assisted implementation** | ~21 weeks |
| **Two engineers** | ~16 weeks |
| **Three engineers** | ~13 weeks (coordination overhead begins to bite) |

**Plan against person-days, not weeks.** Week figures assume a productivity level that varies enormously.

## 17.4 Buffer

| Risk | Buffer |
|---|---|
| Market data harder than expected | +1 week (M2) |
| Classification accuracy below threshold | +1 week (M3) |
| Analysis findings weak, requiring iteration | +2 weeks (M4) |
| Artwork quality requiring brief iteration | +1 week (M7) |
| Marketplace edge cases | +1 week (M9) |

**Recommended plan: 21.5 weeks + 6 weeks buffer = 27.5 weeks to V1.**

Present the buffered figure. A plan that is beaten builds confidence; a plan that slips erodes it.

---

# 18 — Risk Register & Early Warning Signals

## 18.1 Build risks

| Risk | Signal it is happening | Response |
|---|---|---|
| **Thesis fails at M0** | Operator marks most findings "already knew" | Reposition the product before building. This is why M0 exists. |
| **Orchestrator under-built at M1** | Intermittent failures nobody can reproduce; "it worked locally" | Stop feature work. Fix the foundation. Debt here compounds into every later milestone. |
| **Market data inadequate at M2** | Etsy-API-only path produces thin cohorts | Increase collection volume; lower depth expectations; lean harder on measured-only analysis |
| **Classification accuracy low at M3** | Golden-set accuracy below threshold | Fix before M4. Poor classification means confident findings about unmeasured attributes. |
| **Findings weak at M4** | The M4 checkpoint fails | **Stop.** Diagnose whether it is data volume, thresholds, or the thesis. Do not build on top. |
| **Concepts generic at M5** | Concepts read as pastiche; operator selects fewer than three of twenty | Grounding context is too thin or too abstract. Iterate on it before proceeding. |
| **Legal gate incomplete at M6** | Bypass test passes | The gate is decorative. Fix immediately — this is the risk that ends businesses. |
| **Artwork unusable at M7** | Fewer than half of variants pass the gate first time | Brief iteration. If it persists, evaluate an alternative image provider. |
| **Duplicates at M9** | Any duplicate under chaos testing | Do not ship. Four layers exist precisely to make this impossible. |
| **Scope creep throughout** | Features appearing that are not in V1 scope | Every addition requires stating what it displaces |

## 18.2 The four checkpoints

Formal go/no-go decisions, not status updates.

| After | Question | If no |
|---|---|---|
| **M0** | Does the analysis produce non-obvious, actionable findings? | Reposition or stop |
| **M4** | Does it still, at production quality on a fresh niche? | Diagnose before building further |
| **M7** | Is generated artwork usable with less effort than designing from scratch? | Reposition as research-only |
| **M10** | Does the creator prefer this to their previous process? | V1 is not finished |

## 18.3 What to cut if the schedule slips

**In order.** Cut from the top.

| Cut first | Why it is safe |
|---|---|
| Vectorisation | Raster output is publishable |
| Interaction-effect detection | Second-order insight |
| Crowded-loser detection | A refinement of failure analysis |
| Run comparison over time | Requires history to be useful anyway |
| Global search | Convenience |
| Report export | Convenience |
| Concept expansion | The base twenty suffice |
| Manual concept entry | Narrow use case |
| Deep research depth | Standard depth delivers the value |

| **Never cut** | Why |
|---|---|
| **Legal gate** | Guards an existential risk; far harder to retrofit |
| **Design evaluation gate** | Without it there is no automation |
| **Outcome recording** | Data not captured is permanently lost |
| **Baseline display on findings** | Without it, findings mislead |
| **Evidence suppression thresholds** | Without them, noise is presented as insight |
| **Duplicate prevention** | The worst failure the system can cause |
| **Margin floor enforcement** | Prevents unintended loss-making |

## 18.4 The single most likely way this build goes wrong

**Building the reports before the orchestrator is solid.**

Reports are visible, satisfying and demo well. The orchestrator is invisible and demos as a progress bar. The pull toward the former is strong and the consequence is that every later milestone inherits unreliability that surfaces as intermittent, unreproducible failures — the most expensive category of bug to fix, discovered at the worst time.

**Milestone 1 exists to prevent this. Do not compress it.**

---

## Document control

| | |
|---|---|
| **Phase** | 1D of the POD Intelligence specification series |
| **Covers** | Build sequence, milestone definitions, task breakdown, exit criteria, local environment, testing gates, estimates, build risks |
| **Scope** | Version 1 — the personal tool. V2 and V3 entry conditions are in Phase 1C §16. |
| **Excludes** | Application code |
| **Next** | Implementation — beginning with Milestone 0 |
| **Status** | Ready for engineering review |
