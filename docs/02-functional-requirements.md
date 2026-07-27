# 02 — Functional Requirements

**Version:** 1.0 · Numbering is stable and referenced by tests, tickets and the roadmap.

Convention:
- **MUST** — Phase 1 blocking.
- **SHOULD** — Phase 1 target, may slip one phase.
- **MAY** — deferred/optional.
- Each requirement carries **AC** (acceptance criteria) where behaviour is non-obvious.

---

## FR-000 — Cross-cutting

| ID | Requirement |
|---|---|
| FR-001 | Every persisted entity **MUST** carry `workspace_id`, `created_at`, `updated_at`, and (where user-editable) `created_by_user_id`. |
| FR-002 | Every AI-derived output **MUST** persist: prompt template id + version, model tier + resolved model id, input hash, raw response reference, token counts, cost, latency. |
| FR-003 | Every numeric score **MUST** persist the `scoring_config_version` used to produce it. |
| FR-004 | Every external API call **MUST** be recorded in `provider_calls` with provider, endpoint, status, latency, cost, and correlation to the owning `run_step`. |
| FR-005 | All monetary values **MUST** be stored as `bigint` minor units with an ISO-4217 `currency` column. Display formatting is a UI concern. |
| FR-006 | All estimated market metrics (sales, revenue) **MUST** be labelled as estimates in the UI and carry a `confidence` enum (`low`/`medium`/`high`) and `source`. |
| FR-007 | The system **MUST** support cancelling any in-flight run; cancellation stops further spend within 10 seconds and preserves completed step outputs. |
| FR-008 | The system **MUST** support resuming a failed run from the first failed step without repeating successful steps. |
| FR-009 | Every long-running operation **MUST** stream progress to the client (per-step status, percentage, current activity, accumulated cost). |
| FR-010 | The system **MUST** enforce a per-run cost budget; when 80% is consumed the operator is warned, at 100% the run pauses and awaits explicit continuation. |
| FR-011 | All destructive actions (delete run, delete artwork, disconnect integration) **MUST** require confirmation and **MUST** be soft-deletes with a 30-day recovery window. |
| FR-012 | The system **MUST** provide a global search across niches, runs, concepts, artwork and listings. |

---

## FR-100 — Step 1: Research Input

| ID | Req | Detail |
|---|---|---|
| FR-101 | MUST | Operator can start a research run by supplying: **niche** (free text, 2–80 chars) and **product type** (enum from `product_types`). |
| FR-102 | MUST | Optional **style preference**, one of: `vintage`, `typography`, `hand_drawn`, `illustration`, `humour`, `modern`, `auto`. Default `auto`. |
| FR-103 | MUST | `auto` **MUST** cause the pipeline to determine the best-performing style empirically from Step 4 output and record the choice with reasoning. |
| FR-104 | MUST | Optional **depth**: `quick` (5 shops, no visual analysis, no gap engine), `standard` (10 shops, full), `deep` (20 shops, full + extended sub-niche discovery). Default `standard`. |
| FR-105 | MUST | Optional **budget cap** in minor units; defaults from workspace settings. |
| FR-106 | MUST | Niche input **MUST** be normalised (trim, collapse whitespace, lowercase for matching) and deduplicated against existing `niches` by trigram similarity ≥ 0.85, offering to reuse the existing niche. |
| FR-107 | SHOULD | The wizard **SHOULD** show a pre-flight estimate: expected duration, expected cost, expected data sources used. |
| FR-108 | MUST | Starting a run **MUST** be idempotent per `(workspace_id, niche_id, product_type_id, style, depth, idempotency_key)` within a 60-second window. |
| FR-109 | SHOULD | Operator can supply optional **seed keywords** and **excluded terms** that constrain research and SEO. |
| FR-110 | MAY | Operator can start a run from a saved template. |

**AC-100:** Given a niche "Gardening" already exists and the operator types "gardening ", the wizard offers the existing niche and does not create a duplicate row.

---

## FR-200 — Step 2: Opportunity Report Engine

| ID | Req | Detail |
|---|---|---|
| FR-201 | MUST | Compute five sub-scores, each normalised to 0–100: **Demand**, **Competition**, **Trend**, **Profitability**, **Seasonality**. |
| FR-202 | MUST | Compute an **Overall Opportunity Score** 0–100 as a weighted combination defined by the active `scoring_config`. Default weights: Demand 0.30, Competition 0.25 (inverted — lower competition scores higher), Trend 0.20, Profitability 0.15, Seasonality 0.10. |
| FR-203 | MUST | Map the overall score to a verdict band: 0–29 `avoid`, 30–49 `marginal`, 50–69 `good`, 70–84 `strong`, 85–100 `exceptional`. |
| FR-204 | MUST | Every sub-score **MUST** persist its input features (the raw metrics), the normalisation applied, and a human-readable explanation. |
| FR-205 | MUST | Discover and rank **sub-niches** (minimum 5, target 8–15) with, for each: name, description, rationale, estimated demand, estimated competition, sub-niche opportunity score, and example search terms. |
| FR-206 | MUST | Sub-niche discovery **MUST** combine (a) LLM domain expansion, (b) keyword co-occurrence from observed competitor listing titles/tags, and (c) Etsy taxonomy/related-search signals where available. Each sub-niche records which sources supported it. |
| FR-207 | MUST | Seasonality **MUST** be expressed as a 12-point monthly index (100 = annual mean) plus `peak_months`, `trough_months`, and a `seasonality_strength` coefficient of variation. |
| FR-208 | MUST | Profitability **MUST** be computed from real Printify blueprint costs + Etsy fee model + observed competitor price distribution, not from an LLM guess. |
| FR-209 | MUST | If a required data source is unavailable, the affected sub-score is returned with `confidence = low`, a `degraded = true` flag, and the fallback method used; the run does not fail. |
| FR-210 | SHOULD | Produce a natural-language executive summary (≤ 250 words) explaining the verdict, the single biggest risk, and the single biggest opportunity. |
| FR-211 | SHOULD | Compare against previously-researched niches: percentile rank of this opportunity score within the workspace's history. |
| FR-212 | MUST | Report is persisted immutably; re-running creates a new report version, and the operator can diff versions. |

**AC-200:** All five sub-scores plus the overall score are reproducible: re-executing scoring against the stored feature set with the same config version yields byte-identical values.

**Formulas:** see [Appendix A](appendix/a-scoring-models.md) §2.

---

## FR-300 — Step 3: Competitor Analysis Engine

### 3a. Shop discovery & selection

| ID | Req | Detail |
|---|---|---|
| FR-301 | MUST | Discover candidate Etsy shops matching the (niche, product type) via the configured market data provider chain. |
| FR-302 | MUST | Select up to **20** shops. If fewer than 20 qualify, fall back to **10**, then **5**. If fewer than 5, proceed with what exists and mark the run `degraded` with an explicit warning. |
| FR-303 | MUST | Selection score **MUST** combine: estimated monthly sales, estimated monthly revenue, review count, review velocity, active listing count, and shop age — with an explicit **preference for shops under 3 years old** (age multiplier: <1y ×1.15, 1–3y ×1.10, 3–5y ×1.0, >5y ×0.85). |
| FR-304 | MUST | Minimum qualification thresholds (configurable): ≥ 50 lifetime sales OR ≥ 25 reviews; ≥ 5 active listings relevant to the niche; shop not marked closed/on holiday. |
| FR-305 | MUST | Persist the selection rationale per shop (which criteria it passed, its selection score, its rank). |
| FR-306 | MUST | Deduplicate shops already analysed within the last 14 days; reuse cached snapshots rather than re-fetching. |

### 3b. Listing collection

| ID | Req | Detail |
|---|---|---|
| FR-310 | MUST | For each selected shop, collect **all relevant listings** — those whose product type matches and whose title/tags match the niche or any discovered sub-niche. |
| FR-311 | MUST | Cap collection at 250 listings per shop and 3,000 per run (configurable); when capped, select the highest-signal listings (by estimated sales) and record the truncation. |
| FR-312 | MUST | Per listing, capture: `etsy_listing_id`, title, description, tags[], price + currency, shipping cost, free-shipping flag, image count, image URLs, thumbnail reference, product type, listing creation date (age), shop age at capture, bestseller flag, review count, rating, review velocity (reviews/month over trailing 90 days), estimated sales, estimated revenue, favourites, views (where available), section, and personalisation flag. |
| FR-313 | MUST | Store each capture as an immutable `listing_snapshot` row keyed by `(listing_id, captured_at)`, enabling time-series analysis and re-analysis without re-fetching. |
| FR-314 | MUST | Download and store the primary thumbnail (and up to 3 additional images) in object storage, with content hash deduplication. |
| FR-315 | MUST | Record `data_source` and `confidence` per field group (e.g. sales estimate from EverBee CSV = medium confidence; price from Etsy API = high). |

### 3c. Visual & textual style extraction

| ID | Req | Detail |
|---|---|---|
| FR-320 | MUST | Extract a **colour palette** from each thumbnail deterministically (k-means in CIELAB, k=6), producing hex values, proportions, and a mapped **palette family** (e.g. `muted_earth`, `high_contrast_mono`, `pastel`, `neon`, `vintage_washed`, `jewel`, `monochrome_dark`, `natural_neutral`). |
| FR-321 | MUST | Classify **typography style** via a vision model into a controlled vocabulary: `vintage_serif`, `condensed_sans`, `script`, `handwritten`, `slab`, `display_bold`, `retro_groovy`, `minimal_sans`, `distressed`, `none`. |
| FR-322 | MUST | Classify **layout archetype**: `centred_stack`, `badge_circle`, `arched_text`, `left_aligned_block`, `illustration_only`, `text_over_illustration`, `split`, `border_frame`, `repeat_pattern`. |
| FR-323 | MUST | Classify **mockup/presentation style**: `flat_lay`, `model_lifestyle`, `ghost_mannequin`, `hanging`, `folded`, `studio_plain`, `graphic_only`, `in_situ_scene`. |
| FR-324 | MUST | Extract design-content attributes: subject matter tags, humour type (`pun`, `sarcasm`, `wholesome`, `none`), text presence, text length band, and personalisation indicator. |
| FR-325 | MUST | Persist all of the above as a `style_profile` row linked to the listing snapshot, with the extraction model version. |
| FR-326 | SHOULD | Generate and store an image embedding per thumbnail (`vector(512)` in `pgvector`) for visual similarity, clustering and originality checking. |
| FR-327 | MUST | Vision extraction **MUST** be batched (≤ 12 images per call) and cached by image content hash — the same image is never analysed twice. |
| FR-328 | MUST | Extract keyword and tag frequency across all collected listings, with TF-IDF weighting against a niche-agnostic baseline corpus. |

**AC-300:** For a run configured at depth `standard`, the system selects exactly 10 shops (or documents why fewer), collects every matching listing, and produces a style profile for ≥ 95% of listings with a stored thumbnail.

---

## FR-400 — Step 4: Success Analysis Engine

| ID | Req | Detail |
|---|---|---|
| FR-401 | MUST | Partition collected listings into performance cohorts using estimated monthly sales normalised by listing age: **top decile**, **top quartile**, **middle**, **bottom quartile**, **bottom decile**. |
| FR-402 | MUST | For every categorical attribute (palette family, typography, layout, mockup style, product colour, humour type, personalisation, free shipping, bestseller) compute: support in the success cohort, support in the baseline, **lift** = success-support ÷ baseline-support, sample size, and a significance measure (two-proportion z-test; χ² for multi-level). |
| FR-403 | MUST | For every numeric attribute (price, image count, title length, tag count, description length, review velocity, listing age) compute: cohort median/IQR, baseline median/IQR, effect size (Cliff's delta), and an optimal band (the decile range maximising success density). |
| FR-404 | MUST | Emit **weighted success factors**, each with: statement, attribute, value, support %, lift, sample size, confidence (`low`/`medium`/`high` from n and p-value), and a weight (0–1) used downstream by the concept generator and predictor. |
| FR-405 | MUST | Suppress any factor with n < 8 in the cohort or p > 0.10; such factors are stored but flagged `insufficient_evidence` and excluded from downstream weighting. |
| FR-406 | MUST | Produce phrasing exactly matching the product spec style: *"84% of top-decile listings use muted green palettes (lift 2.1×, n=42, high confidence)."* |
| FR-407 | MUST | Compute a **correlation matrix** across numeric attributes and performance, using Spearman rank correlation, and flag multicollinearity between attributes. |
| FR-408 | MUST | Produce a **synthesis card**: "the median winning listing" — modal palette family, modal typography, modal layout, median price, median image count, modal mockup style — rendered as a spec sheet. |
| FR-409 | MUST | When `style = auto`, select the winning style by highest weighted lift among style-bearing attributes and persist the decision with its evidence. |
| FR-410 | SHOULD | Generate visual report assets: palette gallery ranked by success weight, typography distribution bar chart, price-vs-sales scatter with cohort colouring, image-count histogram overlaid by cohort. |
| FR-411 | MUST | Guard against survivorship framing: every factor **MUST** display baseline support alongside cohort support so lift is visible, never bare percentages. |
| FR-412 | SHOULD | Detect and report **interaction effects** for the top 5 factors (e.g. vintage typography lifts only when combined with muted palettes). |

**AC-400:** No factor is presented to the user without cohort support, baseline support, lift, n and confidence. A factor derived from n=3 never appears in the ranked list.

---

## FR-500 — Step 5: Failure Analysis Engine

| ID | Req | Detail |
|---|---|---|
| FR-501 | MUST | Define the failure cohort as listings older than 90 days in the bottom quartile of age-normalised estimated sales, excluding listings with insufficient exposure data. |
| FR-502 | MUST | Apply the identical statistical treatment as FR-402/403 in the inverse direction, producing **anti-factors** with statement, support, lift (>1 means over-represented in failures), n and confidence. |
| FR-503 | MUST | Analyse the same attribute set as Step 4 plus: description quality signals (length, keyword stuffing ratio, readability), tag utilisation (< 13 tags used), title keyword-stuffing, missing size chart/attributes, single-image listings, and price outliers (top/bottom 5% of niche). |
| FR-504 | MUST | Produce two paired outputs: **"Success Factors"** and **"Avoid These Factors"**, with an explicit conflict-resolution rule when an attribute appears in both (report net lift and mark as `ambiguous`). |
| FR-505 | MUST | Distinguish **causal-plausible** anti-factors (e.g. one image, no tags) from **correlational-only** ones, and label them. The UI must not imply causation where none is established. |
| FR-506 | SHOULD | Identify "crowded loser" patterns — attribute combinations that are simultaneously very common and heavily represented in the failure cohort (i.e. what everyone does that doesn't work). |
| FR-507 | MUST | Feed anti-factors into the concept generator as **negative constraints** and into the Opportunity Scoring Engine as penalty terms. |

---

## FR-600 — Step 6: Market Gap Engine

| ID | Req | Detail |
|---|---|---|
| FR-601 | MUST | Build a **coverage matrix** of sub-niche × design angle × style, counting observed competitor listings per cell (supply) . |
| FR-602 | MUST | Estimate **demand** per sub-niche from: search-term signals, aggregate sales in that sub-niche, review velocity, and trend direction. |
| FR-603 | MUST | Compute a **Gap Opportunity Score** 0–100 per cell as a function of demand, inverse supply, monetisability (observed price power), and feasibility (can this be expressed as a printable graphic). Formula in Appendix A §5. |
| FR-604 | MUST | Rank all gaps and return the top N (default 15) with: sub-niche, angle, style, demand evidence, supply evidence, score, and a plain-English explanation of the opportunity. |
| FR-605 | MUST | Exclude cells with demand below a floor (`min_demand_index`, default 20) to avoid recommending genuinely dead markets as "uncontested". This is the single most important guard in this engine. |
| FR-606 | MUST | Produce the demand-vs-supply bubble map data (x = supply, y = demand, size = monetisability, colour = score). |
| FR-607 | SHOULD | Suggest 2–4 concrete design angles per top gap, phrased as creative directions rather than finished concepts. |
| FR-608 | SHOULD | Flag gaps that are gaps *for a reason* — e.g. trademark-heavy territory, seasonal dead zone, or physically unprintable — using the legal term blocklist and a feasibility check. |

**AC-600:** Every returned gap displays both the demand evidence and the supply count. A gap with supply 0 and demand below the floor is never returned.

---

## FR-700 — Step 7: Opportunity Scoring Engine

| ID | Req | Detail |
|---|---|---|
| FR-701 | MUST | Score every concept (and later, every artwork) on five dimensions 0–100: **Market Fit**, **Originality**, **Conversion**, **Competition**, **Opportunity**. |
| FR-702 | MUST | **Market Fit** = weighted alignment between the concept's declared attributes and the success factors from Step 4, minus alignment with anti-factors from Step 5. Deterministic. |
| FR-703 | MUST | **Originality** = 1 − max cosine similarity between the concept's text embedding and the embeddings of collected competitor listing titles/descriptions, combined (once artwork exists) with visual embedding distance to competitor thumbnails. Deterministic. |
| FR-704 | MUST | **Conversion** = model of expected click-through and purchase propensity from attributes empirically associated with conversion (image count plan, mockup style, price band, title structure, personalisation). Starts as an expert-weighted linear model; replaced by a fitted model once ≥ 200 outcomes exist (FR-1610). |
| FR-705 | MUST | **Competition** = inverse density of directly comparable competitor listings (same sub-niche, same angle, similar style), normalised. |
| FR-706 | MUST | **Opportunity** = the Gap Opportunity Score of the cell this concept occupies, or the sub-niche opportunity score if it is not a gap play. |
| FR-707 | MUST | **Opportunity Score** = weighted combination (default: Market Fit 0.30, Conversion 0.25, Opportunity 0.20, Originality 0.15, Competition 0.10), 0–100, with band mapping (`weak` <40, `moderate` 40–59, `promising` 60–74, `strong` 75–87, `exceptional` ≥88). |
| FR-708 | MUST | Provide **reasoning** per dimension: which factors contributed positively and negatively, with their weights. The reasoning is generated from the computed contribution vector, not free-form LLM narrative. |
| FR-709 | MUST | Persist all sub-scores, the contribution vector, the config version, and the feature snapshot, so predictions can be back-tested. |
| FR-710 | MUST | Re-score artwork after generation (originality and conversion change once the image exists) and store both `predicted_at_concept` and `predicted_at_artwork` values. |
| FR-711 | SHOULD | Expose a calibration view comparing predicted scores to realised outcomes once performance data exists. |

**AC-700:** Two concepts with identical attributes produce identical scores. Changing the scoring config version changes scores only for new predictions; historical predictions remain intact and re-scorable on demand.

---

## FR-800 — Step 8: AI Design Generation (Concepts)

| ID | Req | Detail |
|---|---|---|
| FR-801 | MUST | Generate exactly **10 success-derived concepts** grounded in the weighted success factors and the winning style. |
| FR-802 | MUST | Generate exactly **10 gap-derived concepts** grounded in the top-ranked market gaps. |
| FR-803 | MUST | Each concept **MUST** contain: `name`, `description` (40–120 words), `target_audience`, `style`, `design_angle`, `sub_niche`, `visual_direction` (palette family, typography class, layout archetype), `text_content` (if any), `origin` (`success` \| `gap` \| `manual`), `reasoning` (why this should work, citing specific factors/gaps), and the five predictor scores. |
| FR-804 | MUST | Concepts **MUST NOT** reference any competitor shop, listing, brand or artwork. Prompts pass only aggregate statistical attributes. |
| FR-805 | MUST | Deduplicate concepts within a run: reject any concept whose text embedding cosine similarity to another concept in the same run exceeds 0.92; regenerate to fill the quota. |
| FR-806 | MUST | Deduplicate against the workspace's concept history (similarity > 0.95 → flag as near-duplicate of concept X, do not silently drop). |
| FR-807 | MUST | Present concepts to the operator as a selectable board; **artwork generation MUST NOT begin until the operator explicitly selects concepts**. |
| FR-808 | MUST | Support regenerating a single concept (with an optional steer instruction) or the whole set, preserving unselected/selected state. |
| FR-809 | SHOULD | Support manual concept entry, which then flows through the predictor and legal engine identically. |
| FR-810 | SHOULD | Support "more like this" expansion from a selected concept, producing 5 variants along a named axis (tone, audience, complexity). |
| FR-811 | MUST | Every concept records the exact prompt version, factor set and gap set used, so generation is auditable. |

---

## FR-900 — Step 9: Legal & Safety Engine

| ID | Req | Detail |
|---|---|---|
| FR-901 | MUST | Run **before** any artwork generation. Artwork generation for an unscreened or blocked concept **MUST** be impossible at the service layer, not merely hidden in the UI. |
| FR-902 | MUST | Extract candidate protected entities from concept name, description and text content: brand names, character names, celebrity names, sports teams, slogans, film/TV/music titles, and common trademarked phrases. |
| FR-903 | MUST | Check extracted terms against: (a) an internal blocklist of high-risk terms and known-litigious marks, (b) live trademark registries — USPTO, EUIPO, UKIPO — for exact and fuzzy matches within relevant Nice classes (esp. 25 apparel, 21 housewares, 16 paper goods), (c) Etsy's prohibited/restricted items and IP policy term list. |
| FR-904 | MUST | Assess copyright risk: derivative-work indicators, recognisable character/likeness descriptions, quoted lyrics or film dialogue, and use of protected photographic subjects. |
| FR-905 | MUST | Produce a per-concept `risk_level`: `none`, `low`, `medium`, `high`, `blocked`, with: matched terms, matched registrations (registration number, owner, class, status, jurisdiction, link), risk category, and rationale. |
| FR-906 | MUST | `blocked` concepts cannot proceed under any circumstance. `high` requires an explicit operator override with a typed acknowledgement and is recorded in the audit log. `medium` requires acknowledgement. `low`/`none` pass. |
| FR-907 | MUST | For every flagged concept, generate **safer alternatives** — reworded or re-angled versions that preserve the commercial intent without the risky element — and re-screen them automatically. |
| FR-908 | MUST | Screen generated **artwork** as well as concepts: a vision check for logos, recognisable characters, and text that was not in the approved brief. |
| FR-909 | MUST | Screen final **listing text** (title, tags, description) for restricted Etsy terms and trademark terms before draft creation. |
| FR-910 | MUST | Persist an immutable legal screening record per concept and per artwork, retained for at least 7 years, including registry response payloads. |
| FR-911 | MUST | Display a persistent disclaimer: the engine reduces risk but is not legal advice and does not guarantee non-infringement. |
| FR-912 | SHOULD | Cache registry lookups by normalised term for 30 days to control cost and latency. |
| FR-913 | SHOULD | Maintain a workspace-level "burned terms" list of anything that previously triggered a takedown, permanently blocked thereafter. |

**AC-900:** An attempt to call the artwork generation service with a concept whose latest screening is `blocked`, `high`-without-override, or absent, returns `409 LEGAL_GATE_NOT_PASSED` and generates no external API call.

---

## FR-1000 — Step 10: Artwork Generation

### 10a. Brief

| ID | Req | Detail |
|---|---|---|
| FR-1001 | MUST | For each approved concept, generate a **detailed artwork brief** containing: subject description, composition and layout, palette (explicit hex list derived from the success-weighted palette family), typography direction (class, weight, case, arrangement — never a specific licensed font file), texture/finish, background requirement (transparent), aspect ratio and target print dimensions, negative constraints (no logos, no photorealistic faces, no gradients that band, no thin strokes < 2 px at print size), and a POD-suitability note. |
| FR-1002 | MUST | The brief is persisted and editable by the operator before generation. |
| FR-1003 | MUST | The brief is compiled into a provider prompt by the Ideogram adapter, separating semantic content from provider-specific parameters (style preset, magic prompt, aspect ratio, negative prompt, seed). |

### 10b. Generation

| ID | Req | Detail |
|---|---|---|
| FR-1010 | MUST | Generate artwork via **Ideogram**, defaulting to 4 variants per concept, configurable 1–8. |
| FR-1011 | MUST | Support the required style families: typography-led, vintage/retro, hand-drawn, illustration, modern/minimal, humour. Each maps to a documented parameter+prompt template. |
| FR-1012 | MUST | Persist for each generation: seed, all parameters, prompt text, provider request/response ids, cost, latency, and the resulting asset. |
| FR-1013 | MUST | Store the original provider output immutably; all processing produces new derived assets, never overwrites. |
| FR-1014 | MUST | Retry failed or QA-rejected generations up to a configurable limit (default 2) with adjusted parameters, and count retries against the run budget. |

### 10c. Post-processing

| ID | Req | Detail |
|---|---|---|
| FR-1020 | MUST | **Background removal** producing a true alpha-channel transparent PNG, with edge refinement (feather ≤ 1 px, decontamination of halo pixels). |
| FR-1021 | MUST | **Upscale** to meet print requirements: ≥ 300 DPI at the target print area (e.g. 4500×5400 px for a 15"×18" DTG print area). |
| FR-1022 | MUST | **Print-readiness QA**, producing a pass/warn/fail report on: effective DPI, alpha channel presence, percentage of semi-transparent pixels, minimum stroke width at print size, colour count, out-of-gamut colours for DTG, presence of pure white on white-garment mockups, edge artefacts, and file size. |
| FR-1023 | MUST | Generate derived renditions: full-resolution print PNG, web preview (1200 px), thumbnail (400 px), and a checkerboard-backed transparency proof. |
| FR-1024 | SHOULD | **Vectorisation** to SVG for typography-led and flat-illustration outputs, using a raster-to-vector tracer with palette quantisation; store SVG alongside PNG and mark `svg_ready = true`. Vectorisation is skipped (with a reason) for photographic or heavily textured outputs. |
| FR-1025 | SHOULD | Auto-crop to content bounds with configurable padding, and centre on the target canvas. |
| FR-1026 | MUST | Every artwork **MUST** carry an originality check: visual embedding distance to (a) all competitor thumbnails in the run and (b) all prior artwork in the workspace. A similarity above 0.94 to any competitor image raises a hard warning and requires acknowledgement. |
| FR-1027 | MUST | Artwork **MUST NOT** be produced from, conditioned on, or seeded with any competitor image. The generation adapter has no code path that accepts a competitor asset as input. |
| FR-1028 | SHOULD | Support operator-uploaded artwork entering the pipeline at this stage (skipping generation but not QA, legal or originality checks). |

**AC-1000:** A generated artwork that fails the DPI check cannot be attached to a product draft; the UI shows the specific failing criterion and offers upscale/regenerate.

---

## FR-1100 — Step 11: Product Recommendation Engine

| ID | Req | Detail |
|---|---|---|
| FR-1101 | MUST | Maintain a catalogue of Printify **blueprints** × **print providers** × **variants** with production cost, available colours, sizes, print areas, and shipping cost by region, synced from the Printify API and cached. |
| FR-1102 | MUST | Recommend and rank product configurations for the chosen product type using three sub-scores: **Demand** (observed competitor sales concentration by garment type/colour), **Competition** (density of the same configuration in the niche), and **Profitability** (margin at the recommended price after Printify cost, Etsy fees, shipping, and VAT treatment). |
| FR-1103 | MUST | Produce a ranked table with: blueprint, print provider, recommended variants (colours/sizes), unit cost, suggested retail price, gross margin, margin %, fulfilment region, average production time, and provider reliability rating. |
| FR-1104 | MUST | Recommend garment colours empirically — the colours that appear most in the success cohort — and validate them against the artwork (e.g. do not recommend a dark garment for artwork whose dominant ink is near-black). |
| FR-1105 | MUST | Model the Etsy fee stack explicitly and configurably: listing fee, transaction fee %, payment processing % + fixed, offsite ads % (if applicable), regulatory operating fee, and VAT/sales tax treatment. |
| FR-1106 | MUST | Enforce a configurable **minimum margin floor** (default 40%); configurations below it are shown but flagged and cannot be auto-selected. |
| FR-1107 | MUST | Support the required product families: shirts (multiple blueprint tiers), hoodies/sweatshirts, mugs (multiple sizes), and posters/art prints (multiple formats and paper types). |
| FR-1108 | SHOULD | Recommend a **free shipping** strategy with a price-inclusive calculation, since free shipping is an observable success factor in most Etsy niches. |
| FR-1109 | SHOULD | Recommend the number and type of listing images to produce (informed by FR-403's optimal image-count band). |

---

## FR-1200 — Step 12: SEO Engine

| ID | Req | Detail |
|---|---|---|
| FR-1201 | MUST | Generate exactly **10 listing variations** per product draft. |
| FR-1202 | MUST | Each variation contains: `title` (≤ 140 chars), `description` (structured: hook, details, materials/care, sizing, shipping, gift angle, keyword paragraph), `tags` (exactly 13, each ≤ 20 chars, no duplicates, no single-word waste where a phrase fits), `keywords` (ranked with rationale), `positioning` (one-line market stance), and `materials`/`attributes` suggestions. |
| FR-1203 | MUST | Ground keyword selection in run data: observed high-performing competitor tags weighted by TF-IDF and by the sales of the listings using them, plus long-tail terms from sub-niche discovery. Each keyword records its evidence. |
| FR-1204 | MUST | Enforce Etsy constraints as hard validators; a variation failing validation is auto-repaired once, then regenerated. |
| FR-1205 | MUST | Differentiate the 10 variations along declared axes — e.g. gift-focused, audience-focused, humour-led, benefit-led, occasion-led, long-tail-specific, broad-reach, seasonal, personalisation-led, premium-positioned — and label each with its axis. |
| FR-1206 | MUST | Score each variation on: keyword coverage, front-loading of the primary keyword, readability, tag diversity, and estimated competition of its primary keyword. Rank them. |
| FR-1207 | MUST | Allow regeneration of all variations or a single variation, with an optional steer, preserving operator edits elsewhere. |
| FR-1208 | MUST | Allow full manual editing with live validation and character counters. |
| FR-1209 | MUST | Screen all generated text through the Legal Engine's term checks (FR-909) before it can be submitted to Etsy. |
| FR-1210 | SHOULD | Detect and avoid keyword stuffing (repeat-term ratio threshold) and flag titles that read as machine-generated. |
| FR-1211 | SHOULD | Suggest the listing's Etsy taxonomy category and attributes automatically. |

---

## FR-1300 — Step 13: Printify Integration

| ID | Req | Detail |
|---|---|---|
| FR-1301 | MUST | Upload the print-ready artwork to Printify, storing the returned image id and deduplicating by content hash to avoid re-upload. |
| FR-1302 | MUST | Create a Printify product from: blueprint id, print provider id, selected variants, print area placement (position, scale, angle), and title/description. |
| FR-1303 | MUST | Configure per-variant pricing from the pricing engine output. |
| FR-1304 | MUST | Retrieve generated **mockups**, store them in object storage, and associate them with the product draft in a chosen display order. |
| FR-1305 | MUST | Handle Printify's asynchronous behaviours (mockup generation delay, publish lifecycle) via polling with backoff and/or webhook, never blocking a request thread. |
| FR-1306 | MUST | Respect Printify rate limits with a token bucket and exponential backoff on 429. |
| FR-1307 | MUST | Persist the full Printify product payload and id for reconciliation, and support updating an existing product rather than duplicating. |
| FR-1308 | MUST | Detect and surface Printify validation errors (artwork too small, unsupported placement, variant unavailable) as actionable UI messages mapped to remediation. |
| FR-1309 | SHOULD | Support choosing between Printify's own Etsy publish bridge and direct Etsy API creation, configurable per workspace, defaulting to **direct Etsy API** for full SEO control. |
| FR-1310 | MUST | All Printify operations are idempotent via a stored idempotency key per product draft. |

---

## FR-1400 — Step 14: Etsy Integration

| ID | Req | Detail |
|---|---|---|
| FR-1401 | MUST | Authenticate via Etsy Open API v3 OAuth 2.0 with PKCE; store refresh tokens encrypted; refresh proactively before expiry. |
| FR-1402 | MUST | Create listings in **draft** state (`state = draft`). The system **MUST NOT** create an active listing without an explicit operator publish action. |
| FR-1403 | MUST | Set: title, description, price, quantity, taxonomy id, tags, materials, who-made/when-made/is-supply, shipping profile, return policy, section, production partner, and personalisation settings. |
| FR-1404 | MUST | Upload listing images in the operator-defined order, with the primary image first; support up to 10 images and 1 video slot (video not generated in Phase 1). |
| FR-1405 | MUST | Create listing inventory with variant-level pricing and SKUs mapped to Printify variants. |
| FR-1406 | MUST | Respect Etsy rate limits (10 req/s, 10,000/day) via a shared token bucket, reserving a daily quota for publishing and performance sync. |
| FR-1407 | MUST | Handle and surface Etsy validation errors field-by-field, mapped to the originating UI field. |
| FR-1408 | MUST | Store `etsy_listing_id`, state, and last-sync timestamp; support re-sync and drift detection between local and remote state. |
| FR-1409 | MUST | Publishing (draft → active) is a distinct, separately-authorised operation triggered only from the Final Review page. |
| FR-1410 | MUST | On any partial failure (e.g. listing created, images failed), the system records a `partial` status and offers a targeted retry that does not duplicate the listing. |
| FR-1411 | SHOULD | Support updating an existing listing's SEO in place (for later optimisation experiments). |
| FR-1412 | MUST | Token revocation, scope loss and 401 responses **MUST** produce a clear reconnection prompt rather than silent failure. |

---

## FR-1500 — Step 15: Final Review & Publish

| ID | Req | Detail |
|---|---|---|
| FR-1501 | MUST | A single review screen showing: concept summary, artwork (with transparency proof), all mockups, product configuration and variants, pricing table, profit estimate per unit and per variant, full SEO variation selected, legal screening status, and the Opportunity Score. |
| FR-1502 | MUST | Display a **pre-publish checklist** with pass/fail: legal cleared, artwork QA passed, margin ≥ floor, tags = 13, title ≤ 140, ≥ N images, shipping profile set, taxonomy set, originality check passed. Publish is disabled until all hard checks pass. |
| FR-1503 | MUST | The profit estimate **MUST** itemise: retail price, Printify cost, Etsy listing fee, transaction fee, payment processing, offsite ads (worst case), shipping, VAT, and net profit with margin %. |
| FR-1504 | MUST | Publishing requires an explicit confirm dialog restating what will happen and to which shop. |
| FR-1505 | MUST | Record who published, when, and the exact payload sent; make it retrievable in the audit log. |
| FR-1506 | MUST | After publish, transition the entity to `published` and schedule the first performance sync. |
| FR-1507 | SHOULD | Support "Save as draft and review later" and a review queue of pending drafts. |
| FR-1508 | SHOULD | Support publishing in batches with a per-item checklist, still one explicit confirmation per batch. |

---

## FR-1600 — Step 16: Analytics & Learning

### 16a. Storage & history

| ID | Req | Detail |
|---|---|---|
| FR-1601 | MUST | Persist permanently: research runs, opportunity reports, competitor shop/listing snapshots, success reports, failure reports, gaps, concepts, briefs, artwork, product drafts, SEO variations, listings, and all provider calls. |
| FR-1602 | MUST | Every report is versioned and immutable; regeneration creates a new version with a link to its predecessor. |

### 16b. Performance tracking

| ID | Req | Detail |
|---|---|---|
| FR-1605 | MUST | Sync published listing performance from Etsy on a schedule (default daily): views, favourites, orders, revenue, and listing state. |
| FR-1606 | MUST | Store performance as time-series snapshots; never overwrite. |
| FR-1607 | MUST | Compute derived metrics: views/day, conversion rate, revenue/day, days-to-first-sale, and age-normalised performance percentile within the workspace and within the niche. |
| FR-1608 | SHOULD | Ingest Printify order/fulfilment data for true cost reconciliation. |

### 16c. Dashboards

| ID | Req | Detail |
|---|---|---|
| FR-1609 | MUST | Provide dashboards for: portfolio performance, per-niche performance, per-concept-origin performance (success-derived vs gap-derived), prediction accuracy/calibration, cost and spend, and pipeline throughput. |

### 16d. Learning loop

| ID | Req | Detail |
|---|---|---|
| FR-1610 | MUST | Maintain a **feature store** row per published listing linking its design/SEO/pricing attributes to its realised performance. |
| FR-1611 | MUST | Provide a recalibration job that fits attribute weights against realised outcomes (regularised regression initially; gradient-boosted trees when n ≥ 500) and proposes a **new scoring config version**. |
| FR-1612 | MUST | New scoring configs are **proposed, not auto-activated**. The operator sees: proposed weight changes, back-test results on historical outcomes (with time-based train/test split), and predicted-vs-actual improvement, then activates or rejects. |
| FR-1613 | MUST | Activation is reversible; the previous config remains and any config can be reinstated. |
| FR-1614 | MUST | Below a minimum outcome count (default 50), recalibration is disabled and the system uses expert priors, stating so in the UI. |
| FR-1615 | SHOULD | Shrink fitted weights toward the expert prior proportionally to sample size (empirical-Bayes style), preventing wild swings from small samples. |
| FR-1616 | SHOULD | Detect and report which success factors were predictive and which were not, closing the loop visibly for the operator. |
| FR-1617 | MAY | Support controlled experiments: publish paired listings differing in one attribute and measure the difference. |

---

## FR-1700 — Integrations management & settings

| ID | Req | Detail |
|---|---|---|
| FR-1701 | MUST | A settings area to connect/disconnect Etsy, Printify, Ideogram, the market data provider, and the AI provider, showing connection health, scopes, quota usage and last successful call. |
| FR-1702 | MUST | Credentials **MUST** be encrypted at rest with envelope encryption and never returned to the client in full (masked display only). |
| FR-1703 | MUST | Configurable settings: default currency, fee model parameters, margin floor, budget caps, run depth default, scoring config selection, retention policy, and notification preferences. |
| FR-1704 | MUST | A cost/usage page showing spend by provider and by run, month-to-date, against configured caps. |
| FR-1705 | SHOULD | Export all workspace data as a portable archive (JSON + assets manifest). |
| FR-1706 | SHOULD | Notification on run completion, run failure, budget threshold, legal block, publish success/failure — in-app, plus email/webhook optionally. |

---

## FR-1800 — Non-happy paths (explicit)

| ID | Requirement |
|---|---|
| FR-1801 | If the market data provider returns zero shops, the run continues with Etsy public data only, produces a `degraded` opportunity report, and clearly explains what is missing and what to do about it. |
| FR-1802 | If vision analysis fails for a listing, that listing is retained with a null style profile and excluded from style statistics, with the exclusion count reported. |
| FR-1803 | If an LLM returns schema-invalid output, the system repairs once, retries once with a stricter prompt, then fails that step with the raw output preserved for inspection. |
| FR-1804 | If Ideogram fails or is unavailable, the artwork step is marked blocked-external and the concept remains queued; no other step is affected. |
| FR-1805 | If Etsy rejects a draft, the product draft retains all local state and the operator can fix and retry without regenerating anything. |
| FR-1806 | If a run exceeds its budget mid-step, the current step completes (to avoid wasted spend) and the run pauses before the next step. |
| FR-1807 | If a duplicate publish is attempted, idempotency keys prevent a second Etsy listing being created. |
| FR-1808 | If the operator's Etsy token is revoked, all pending publish operations move to `needs_reauth` and are resumable after reconnection. |
