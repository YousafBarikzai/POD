# Appendix C — Glossary

**Version:** 1.0 · Terms are used with exactly these meanings throughout the specification, in code identifiers, and in the UI.

---

## Domain entities

| Term | Definition |
|---|---|
| **Workspace** | The tenancy boundary. Owns all data. Exactly one exists in Phase 1. Every table carries `workspace_id`. |
| **Niche** | A market under investigation, e.g. "Gardening". Persistent across runs; carries status `active` / `rejected` / `exhausted`. |
| **Sub-niche** | A discovered segment within a niche, e.g. "Composting". Independently scored and rankable. |
| **Product Type** | A physical product category (T-Shirt, Hoodie, Mug, Poster, Tote, Sweatshirt). Maps to Printify blueprints. |
| **Research Run** | One execution of the pipeline for a (niche, product type, style, depth) combination. The unit of work, cost and audit. |
| **Run Step** | One node in the run's DAG. The unit of retry, resume, timing and spend. |
| **Depth** | Run scope: `quick` (5 shops), `standard` (10), `deep` (20). |
| **Competitor Shop** | An observed Etsy shop. An identity record; volatile facts live in snapshots. |
| **Competitor Listing** | An observed Etsy listing. An identity record. |
| **Listing Snapshot** | An immutable, timestamped observation of a listing's volatile facts (price, sales estimate, reviews, images). |
| **Style Profile** | The extracted visual fingerprint of a listing image: palette, palette family, typography class, layout archetype, mockup style, subject tags, embedding. |
| **Concept** | A named, described design idea with audience, style, visual direction, reasoning and predicted scores. **Text only — no image exists yet.** |
| **Artwork** | A rendered, background-removed, print-validated image asset derived from a concept via a brief. |
| **Artwork Brief** | The specification handed to the image provider: subject, composition, palette hexes, typography direction, constraints, print dimensions. |
| **Product Draft** | The convergence entity: concept + artwork + blueprint + variants + pricing + SEO, pre-publication. |
| **Listing** | A draft or published Etsy listing owned by the operator, tracked over time. |
| **Performance Snapshot** | An immutable, timestamped capture of a listing's views, favourites, orders and revenue. |
| **Scoring Config** | A versioned set of weights, normalisation parameters and thresholds. Exactly one is active per workspace. |
| **Outcome Feature Row** | The learning loop's training record: a listing's design/SEO/pricing attributes joined to its realised performance. |

---

## Analytical terms

| Term | Definition |
|---|---|
| **Cohort** | A percentile-defined group of listings by age-normalised performance: `top_decile`, `top_quartile`, `middle`, `bottom_quartile`, `bottom_decile`. |
| **Age-normalised performance** | `estimated monthly sales ÷ min(listing age in months, 12)`. Prevents old listings from appearing successful purely through accumulated time. |
| **Support** | The proportion of a group exhibiting an attribute value. Always reported for both the cohort and the baseline population. |
| **Baseline** | The full observed population of the run. Every factor's support is reported against it, so lift is always visible. |
| **Lift** | `cohort support ÷ baseline support`. A lift of 2.1× means the attribute is 2.1 times more common among winners than in the market generally. **The number that matters** — a bare percentage is meaningless without it. |
| **Success Factor** | A statistically qualified statement that an attribute correlates with high performance, carrying support, baseline, lift, n, p-value, confidence and weight. |
| **Anti-Factor** | The same, for the failure cohort. Carries a penalty weight and a causality label. |
| **Insufficient evidence** | A factor with n < 8 or p > 0.10. Stored and visible but excluded from ranking and from downstream weighting. |
| **Ambiguous factor** | An attribute appearing in both the success and failure reports. Reported with net lift and down-weighted to 40%. |
| **Causal-plausible** | An anti-factor with a mechanistic explanation (one image, no tags). Distinguished in the UI from `correlational-only`. |
| **Crowded loser** | An attribute combination that is both very common and over-represented in failures — what everyone does that doesn't work. |
| **Coverage Cell** | An intersection of (sub-niche × design angle × style) with an observed supply count and an estimated demand index. |
| **Market Gap** | A coverage cell with high demand, low supply, and a Gap Opportunity Score above the ranking threshold. |
| **Demand floor** | The minimum demand index (default 20) below which a cell is excluded from gap ranking. Prevents recommending deserts as opportunities. |
| **Monetisability** | Observed price power in a sub-niche: median price combined with achievable margin. |
| **Synthesis card** | The "median winning listing" spec sheet — modal palette, typography, layout, mockup, median price and image count. |
| **Interaction effect** | Two attributes whose combined lift exceeds the product of their individual lifts. |

---

## Scoring terms

| Term | Definition |
|---|---|
| **Opportunity Score** | 0–100 composite of Demand, Competition (inverted), Trend, Profitability and Seasonality. Answers "is this niche worth entering?" |
| **Verdict band** | `avoid` / `marginal` / `good` / `strong` / `exceptional`, mapped from the Opportunity Score. |
| **Gap Opportunity Score** | 0–100 per coverage cell from demand, inverse supply, monetisability and feasibility. |
| **Design Opportunity Score** | 0–100 composite of Market Fit, Visual Quality, Competition (inverted), Originality, multiplied by POD Suitability. Answers "what is this specific design's estimated potential?" — an evidence-based estimate, never a guarantee. |
| **Market Fit** | Weighted alignment of a concept's attributes with success factors, minus alignment with anti-factors. |
| **Originality** | `1 − max similarity` to competitor listings/images and to the workspace's own prior work. |
| **Conversion** | Modelled purchase propensity from attributes empirically linked to conversion. Expert-weighted initially, fitted from outcomes later. |
| **Contribution vector** | The itemised, signed point contributions producing a score. The UI's reasoning display renders this — it is never free-form model narrative. |
| **Shrinkage** | Blending fitted weights toward expert priors in proportion to sample size (`λ = n/(n+150)`). Prevents small samples from rewriting the model. |
| **Calibration** | How well estimated scores match realised outcomes. Measured by rank correlation, Brier score and precision at the top. |
| **Estimated Potential** | The composite design score. Deliberately named to signal an evidence-based estimate rather than a prediction of guaranteed results. |

---

## Technical terms

| Term | Definition |
|---|---|
| **Orchestrator** | The durable, resumable state machine that executes a run's step DAG with budget enforcement. |
| **Step DAG** | The declarative dependency graph of pipeline steps. Data, not control flow. |
| **Human gate** | A point where the run stops and awaits an explicit operator action: concept selection, legal clearance, publish. |
| **Degraded run** | A run that completed with reduced fidelity because a data source or capability was unavailable. Always visibly labelled. |
| **Provenance** | Per-field record of which provider supplied a value, when, at what confidence, and whether it is an estimate. |
| **Capability** | A declared ability of a market data provider (e.g. `listingSalesEstimates`). Engines adapt to what is available rather than failing. |
| **Adapter** | The isolation boundary around an external provider. No vendor SDK is imported outside its adapter. |
| **Budget guard** | The component that reserves estimated spend before a step and commits actual spend after it. |
| **Idempotency key** | A stable hash guaranteeing a side effect happens at most once despite retries. |
| **Circuit breaker** | Per-provider failure isolation. When open, dependent steps become `blocked_external` rather than failing. |
| **Reaper** | The maintenance job that requeues steps whose worker died mid-execution. |
| **Model tier** | An abstraction over concrete AI models: `reasoning`, `analysis`, `extraction`, `vision`. Code references tiers, never model names. |
| **Prompt registry** | Versioned prompt files with schemas, metadata and eval suites. Prompts are never string literals in application code. |
| **Untrusted data block** | The delimited wrapper isolating external text in a prompt. Content inside is data, never instruction. |
| **Content-hash cache** | Caching keyed by the sha256 of the input (image, term, prompt input), so identical work is never repeated or repaid. |
| **Expand/contract** | The migration discipline: additive change → backfill → dual-write → dual-read → remove, so every schema state works with the deployed application version. |
| **Snapshot table** | An immutable, time-partitioned table. Corrections are new rows; nothing is ever updated. |
| **Feature flag** | A workspace-targetable switch decoupling deployment from release. |

---

## Commerce terms

| Term | Definition |
|---|---|
| **Blueprint** | A Printify product model (e.g. "Gildan 5000 Unisex Heavy Cotton Tee"). |
| **Print Provider** | The factory fulfilling a blueprint. Cost, quality, lead time and available variants all vary by provider. |
| **Variant** | A specific blueprint + provider + colour + size combination, with its own production cost. |
| **Print area** | The printable region on a blueprint, with dimensions and a required DPI. |
| **Placement** | Artwork position, scale and rotation within a print area. |
| **Mockup** | A rendered product image used as an Etsy listing photo. |
| **Fee stack** | The itemised deductions from retail price: production, listing fee, transaction fee, payment processing, offsite ads, shipping, VAT. |
| **Offsite Ads** | Etsy's advertising fee (up to 15% of order value) charged on attributed sales. **Modelled as charged by default**, because optimistic margin maths is how POD sellers lose money. |
| **Margin floor** | The minimum acceptable net margin (default 40%). Configurations below it are shown, flagged, and cannot be auto-selected. |
| **SEO variation** | One of ten generated listing versions, each along a declared differentiation axis. |
| **Differentiation axis** | The angle a variation takes: gift, audience, humour, benefit, occasion, longtail, broad, seasonal, personalisation, premium. |
| **Draft-only automation** | The rule that the system creates Etsy listings in `draft` state and never activates one without an explicit human publish action. |

---

## Legal & safety terms

| Term | Definition |
|---|---|
| **Legal gate** | The mandatory screening that must pass before any artwork generation. Enforced at the service layer, not the UI. |
| **Risk level** | `none` / `low` / `medium` / `high` / `blocked`. Determined by a deterministic rule table, not by a model's opinion. |
| **Nice class** | The international trademark goods/services classification. Classes 25 (apparel), 21 (housewares) and 16 (paper goods) are the relevant ones here. |
| **Override** | An operator decision to proceed past a `medium` or `high` risk. Requires step-up authentication, a typed confirmation and a written justification, all recorded immutably. |
| **Burned term** | A term that previously caused a takedown. Permanently blocked for that workspace thereafter. |
| **Safer alternative** | A reworded or re-angled concept preserving commercial intent without the risky element, generated automatically and re-screened. |
| **Originality check** | Embedding-distance comparison of generated artwork against competitor thumbnails and the workspace's own prior artwork. |

---

## Metrics

| Term | Definition |
|---|---|
| **Review velocity** | Reviews received per month over the trailing 90 days. The most reliable public proxy for current sales momentum. |
| **Review-to-sale ratio** | The assumed number of sales per review (default 22). Used only when sales estimates are unavailable; calibrated against the operator's own realised sales in Phase 5. |
| **Days to first sale** | Days between publish and the first recorded order. A leading indicator of listing quality. |
| **Age-normalised percentile** | A listing's rank within its cohort after adjusting for how long it has been live. The learning loop's target variable. |
| **Estimate** | Any value not directly measured. **Always** labelled in the UI with its source and confidence. |
| **Confidence** | `low` / `medium` / `high`, derived from sample size, p-value and data source — never asserted arbitrarily. |
| **Degradation** | Reduced analytical fidelity from a missing capability. Always surfaced, never silent. |
