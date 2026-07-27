# Appendix A — Scoring Model Specification

**Version:** 1.0 · Every formula here is implemented in `packages/domain/scoring` and `packages/domain/stats` as pure functions, exhaustively unit- and property-tested. No LLM produces any number in this document.

---

## 1. Universal conventions

### 1.1 Normalisation

All raw features are mapped to `[0, 100]` before weighting. Three normalisers are used, selected per feature in `scoring_configs.normalisation`:

```
minmax(x, lo, hi)      = clamp( (x − lo) / (hi − lo), 0, 1 ) × 100
logistic(x, μ, k)      = 100 / (1 + e^(−k(x − μ)))
percentile(x, ref[])   = 100 × (rank of x within ref) / |ref|
```

- `minmax` is used where absolute anchors are meaningful and stable (price, image count).
- `logistic` is used where the response should saturate (competition density, review velocity).
- `percentile` is used where only relative position is meaningful (demand versus a workspace's own history).

**Winsorisation:** all inputs are winsorised at the 2nd and 98th percentile of their reference distribution before normalisation, so a single outlier shop cannot distort a niche's scores.

**Inversion:** features where "more is worse" (competition, saturation) are inverted after normalisation: `inverted = 100 − normalised`.

### 1.2 Confidence

```
confidence(n, p) =
  'high'   if n ≥ 30 and p ≤ 0.01
  'medium' if n ≥ 15 and p ≤ 0.05
  'low'    otherwise
```
Applied to statistical factors. For sub-scores, confidence is the minimum confidence of the features that fed it, degraded one level if any feature used a fallback proxy.

### 1.3 Score bands

| Range | Opportunity verdict | Design Success band |
|---|---|---|
| 0–29 | `avoid` | `weak` (<40) |
| 30–49 | `marginal` | `moderate` (40–59) |
| 50–69 | `good` | `promising` (60–74) |
| 70–84 | `strong` | `strong` (75–87) |
| 85–100 | `exceptional` | `exceptional` (≥88) |

---

## 2. Opportunity Score (Step 2)

```
Opportunity = 0.30·Demand + 0.25·CompetitionInv + 0.20·Trend
            + 0.15·Profitability + 0.10·Seasonality
```
Weights are the default `expert_prior` config; they are the primary target of the learning loop.

### 2.1 Demand Score

Features (each normalised, then weighted):

| Feature | Source | Normaliser | Weight |
|---|---|---|---|
| `total_est_monthly_sales` across observed niche listings | market data | `logistic(μ=1500, k=0.0018)` | 0.30 |
| `median_review_velocity_90d` | snapshots | `logistic(μ=2.5, k=0.9)` | 0.25 |
| `active_listing_engagement` = Σ favourites / Σ listings | Etsy | `minmax(0, 400)` | 0.15 |
| `sales_concentration_inverse` = 1 − HHI of sales across shops | derived | `minmax(0, 1)` | 0.15 |
| `search_term_breadth` = count of distinct high-TFIDF terms | keyword stats | `minmax(20, 400)` | 0.15 |

`sales_concentration_inverse` matters: a niche where one shop takes 80% of sales has high raw demand but is not *available* demand. The Herfindahl-Hirschman Index over shop sales shares captures this directly.

**Fallback when sales estimates are unavailable:** `total_est_monthly_sales` is replaced by `Σ (review_count × review_to_sale_ratio) / listing_age_months`, with `review_to_sale_ratio` defaulting to 22 and calibrated per niche in Phase 5. Confidence drops to `low`.

### 2.2 Competition Score (inverted before use)

| Feature | Normaliser | Weight |
|---|---|---|
| `listings_per_100_monthly_sales` (saturation) | `logistic(μ=45, k=0.06)` | 0.30 |
| `median_shop_age_months` (entrenchment) | `minmax(6, 96)` | 0.20 |
| `top_decile_review_moat` = median reviews of top-decile listings | `logistic(μ=250, k=0.008)` | 0.25 |
| `bestseller_density` = share of listings flagged bestseller | `minmax(0, 0.25)` | 0.15 |
| `price_compression` = 1 − (IQR / median price) | `minmax(0, 1)` | 0.10 |

`price_compression` is a subtle but real signal: when everyone is priced within a narrow band, the niche has commoditised and differentiation on price is closed.

```
CompetitionInv = 100 − Competition
```

### 2.3 Trend Score

| Feature | Method | Weight |
|---|---|---|
| `review_velocity_slope` | OLS slope of monthly review counts over trailing 12 months (from snapshots), normalised by mean | 0.40 |
| `new_listing_rate` | Share of observed listings created in the last 180 days | 0.25 |
| `new_entrant_rate` | Share of observed shops under 12 months old | 0.20 |
| `external_trend_index` | External search-trend series where available | 0.15 |

**Interpretation guard:** a high `new_listing_rate` is ambiguous — it signals both growth and incoming competition. It is therefore weighted positively in Trend but also feeds `saturation` in Competition, so a flood of new listings raises Trend and lowers CompetitionInv, netting close to neutral. This is deliberate: the naive version of this score rewards exactly the niches that are about to become unprofitable.

**Fallback:** with fewer than two snapshot generations, slope cannot be computed; Trend falls back to `new_listing_rate` and `new_entrant_rate` reweighted to 0.6/0.4 with `confidence = low`.

### 2.4 Profitability Score

Computed from **real** Printify costs and the workspace fee model, never estimated by a model.

```
observed_price      = median price of the niche's top-quartile listings
unit_cost           = cheapest viable blueprint variant cost + shipping
net_profit          = feeStack(observed_price, unit_cost, feeModel, offsiteAds = true)
margin_pct          = net_profit / observed_price
price_headroom      = p75_price / median_price
```

| Feature | Normaliser | Weight |
|---|---|---|
| `margin_pct` | `minmax(0.15, 0.60)` | 0.50 |
| `net_profit` absolute | `minmax(200, 1500)` minor units | 0.25 |
| `price_headroom` | `minmax(1.0, 1.8)` | 0.15 |
| `free_shipping_feasible` (margin holds with shipping absorbed) | boolean → 100/40 | 0.10 |

**Note:** margin is evaluated with offsite ads charged. Optimistic margin maths is the most common way POD sellers lose money (doc 14 §9).

### 2.5 Seasonality Score

```
S[m]                 = monthly demand index, m = 1..12, mean-normalised to 100
seasonality_strength = σ(S) / μ(S)                        (coefficient of variation)
months_to_peak       = cyclic distance from now to the highest S[m]
```

| Component | Contribution |
|---|---|
| Evergreen bonus | `100 − 120 × seasonality_strength`, floored at 0 |
| Timing bonus | `+25` if `months_to_peak` ∈ [2, 5] (enough lead time to build and rank), `+10` if ∈ [1, 6], else 0 |
| Peak magnitude | `+15 × minmax(max(S)/100, 1.0, 2.5)` |

```
Seasonality = clamp(evergreen + timing + peak, 0, 100)
```

**Rationale:** an evergreen niche is worth more than a seasonal one of equal magnitude, but a seasonal niche entered 3 months before peak is worth more than the same niche entered 1 month after. The score encodes both.

---

## 3. Shop Selection Score (Step 3)

```
Selection = ageMultiplier × ( 0.30·norm(est_monthly_sales)
                            + 0.25·norm(est_monthly_revenue)
                            + 0.20·norm(review_count)
                            + 0.15·norm(review_velocity_90d)
                            + 0.10·norm(active_listing_count) )
```

```
ageMultiplier = 1.15  if shop_age < 12 months
              = 1.10  if 12 ≤ shop_age < 36 months
              = 1.00  if 36 ≤ shop_age < 60 months
              = 0.85  if shop_age ≥ 60 months
```

**Why prefer young shops** (a product requirement, and a correct one): a shop that reached strong sales in under three years demonstrates a *currently replicable* strategy. A ten-year-old shop's advantage is substantially accumulated authority that a new entrant cannot copy. Young winners are better teachers.

**Qualification gates (applied before scoring):** ≥ 50 lifetime sales OR ≥ 25 reviews; ≥ 5 relevant active listings; not closed or on holiday. Shops failing any gate are excluded and the exclusion is recorded.

**Fallback ladder:** take top 20 → if fewer than 20 qualify, take 10 → if fewer than 10, take 5 → if fewer than 5, proceed and mark the run `degraded`.

---

## 4. Success & Failure Factor Statistics (Steps 4–5)

### 4.1 Cohort assignment

```
performance_metric(listing) = est_monthly_sales,  or the proxy when unavailable
age_normalised(listing)     = performance_metric / min(listing_age_days / 30, 12)
```
Capping the age divisor at 12 months prevents a five-year-old listing from being unfairly diluted.

Cohorts by percentile of `age_normalised` within the run's population, excluding listings younger than 30 days (insufficient exposure):

| Cohort | Percentile |
|---|---|
| `top_decile` | ≥ 90 |
| `top_quartile` | ≥ 75 |
| `middle` | 25–75 |
| `bottom_quartile` | ≤ 25 |
| `bottom_decile` | ≤ 10 |

Success factors use `top_decile` (falling back to `top_quartile` if the decile yields n < 15). Failure factors use `bottom_quartile` restricted to listings ≥ 90 days old.

### 4.2 Categorical attributes

For attribute value `v`:

```
support_cohort   = count(cohort, attr = v) / |cohort|
support_baseline = count(population, attr = v) / |population|
lift             = support_cohort / support_baseline
n                = count(cohort, attr = v)
```

Two-proportion z-test:
```
p̂ = (x_c + x_p) / (n_c + n_p)
z  = (support_cohort − support_baseline) / sqrt( p̂(1−p̂)(1/n_c + 1/n_p) )
p  = 2(1 − Φ(|z|))
```

For multi-level attributes, a χ² test of independence across all levels is computed as an omnibus check before individual levels are reported.

**Suppression:** `n < 8` or `p > 0.10` → stored with `insufficient_evidence = true`, excluded from ranking and from downstream weighting.

### 4.3 Numeric attributes

```
effect_size = Cliff's delta between cohort and non-cohort distributions
optimal_band = argmax over contiguous decile ranges of
               ( count(cohort in band) / count(population in band) )
               subject to count(population in band) ≥ 20
```
Cliff's delta is used rather than Cohen's d because these distributions are heavily skewed and non-normal; a rank-based effect size is the honest choice.

### 4.4 Factor weight

```
weight = clamp( w_lift × w_conf × w_size , 0, 1 )

w_lift = minmax( |ln(lift)| , 0 , ln(3) )          // lift of 3× or 1/3× saturates
w_conf = { high: 1.0, medium: 0.7, low: 0.35 }
w_size = minmax( ln(n) , ln(8) , ln(60) )
```

This weight is what the Opportunity Scoring Engine consumes. A factor with lift 2.1×, n = 42, high confidence yields `weight ≈ 0.87` — matching the worked example in doc 5 §3.6.

### 4.5 Anti-factor penalty weight

Identical computation on the failure cohort, producing `penalty_weight`. When an attribute appears as both a success factor and an anti-factor, `net_lift = lift_success / lift_failure` is reported and the factor is marked `ambiguous`; ambiguous factors receive `weight × 0.4` in downstream use.

---

## 5. Gap Opportunity Score (Step 6)

For a coverage cell (sub-niche × angle × style):

```
demand_index   = 0.5·norm(subniche_est_sales) + 0.3·norm(subniche_review_velocity)
               + 0.2·norm(subniche_term_breadth)

supply_index   = logistic( listings_in_cell , μ = 12 , k = 0.22 ) × 100

monetisability = 0.6·norm(median_price_in_subniche) + 0.4·norm(margin_pct_at_that_price)

feasibility    = 100 − penalties   // trademark-heavy −40, seasonally dead −30,
                                   // unprintable −50, no observable audience −25
```

```
Gap = 0.40·demand_index
    + 0.30·(100 − supply_index)
    + 0.20·monetisability
    + 0.10·feasibility
```

**The demand floor — the single most important guard in this engine:**

```
if demand_index < min_demand_index (default 20):
    the cell is EXCLUDED, not scored
```

Without it, the engine's top recommendation is always a cell nobody has entered because nobody wants it. An "uncontested market" with no demand is not an opportunity; it is a desert. Every returned gap displays both its demand evidence and its supply count so this is visible to the operator (FR-604, AC-600).

---

## 6. Opportunity Score (Step 7)

```
DesignSuccess = 0.30·MarketFit + 0.25·Conversion + 0.20·Opportunity
              + 0.15·Originality + 0.10·CompetitionInv
```

### 6.1 Market Fit

```
positive = Σ over success factors f matched by the concept:  weight(f) × match(f)
negative = Σ over anti-factors a matched by the concept:     penalty_weight(a) × match(a)
maxPos   = Σ over all success factors: weight(f)

MarketFit = clamp( 100 × (positive − negative) / max(maxPos, ε) , 0 , 100 )
```

`match(f) ∈ [0,1]`: 1.0 for an exact categorical match (palette family, typography class, layout archetype), 0.6 for a related-family match, and for numeric bands, the fraction of the concept's intended value falling inside the optimal band.

### 6.2 Originality

```
textSim   = max cosine similarity( concept.embedding , competitor listing embeddings )
visualSim = max cosine similarity( artwork.embedding , competitor thumbnail embeddings )
            // only available at the artwork stage
selfSim   = max cosine similarity( concept.embedding , prior workspace concepts )

Originality(concept stage) = 100 × (1 − max(textSim, 0.85·selfSim))
Originality(artwork stage) = 100 × (1 − max(0.5·textSim + 0.5·visualSim, 0.85·selfSim))
```

Self-similarity is discounted by 0.85 because repeating your own successful direction is a legitimate strategy, whereas repeating a competitor is not.

### 6.3 Conversion

An expert-weighted linear model until ≥ 200 outcomes exist, then refitted (§8).

| Feature | Prior coefficient | Rationale |
|---|---|---|
| Image count in the empirically optimal band | +0.22 | Strongest and most consistent Etsy signal |
| Mockup style matches the winning style in this niche | +0.18 | |
| Price within the top-quartile IQR | +0.16 | Under-pricing signals low quality on Etsy |
| Primary keyword front-loaded in title | +0.12 | |
| Design legible at 200 px thumbnail (from artwork QA) | +0.12 | |
| Personalisation offered where the niche rewards it | +0.08 | |
| Free shipping | +0.07 | |
| Text length in the optimal band | +0.05 | |
| Matches an ambiguous or anti-factor attribute | negative, per penalty weight | |

Output normalised to 0–100 via `logistic` against the niche's fitted distribution.

### 6.4 Competition (inverted)

```
comparable = count of competitor listings sharing (sub_niche, design_angle, style_family)
Competition    = logistic(comparable, μ = 15, k = 0.18) × 100
CompetitionInv = 100 − Competition
```

### 6.5 Opportunity

The Gap Opportunity Score of the cell the concept occupies; for non-gap concepts, the sub-niche opportunity score.

### 6.6 Contribution vector

Every dimension emits the itemised list of contributing factors with their signed point contributions. The UI's reasoning display is a rendering of this vector, not free-form model narrative (FR-708) — which is why the reasoning is always exactly consistent with the score.

---

## 7. Product Recommendation Score (Step 11)

```
Product = 0.35·Demand + 0.25·CompetitionInv + 0.40·Profitability
```

| Sub-score | Computation |
|---|---|
| Demand | Share of the success cohort using this blueprint class and colour family, normalised |
| Competition | Density of this exact blueprint + colour combination in the niche, inverted |
| Profitability | `margin_pct` at the recommended price with offsite ads charged, `minmax(floor, 0.60)` |

Configurations below `margin_floor_pct` are displayed but flagged and cannot be auto-selected (FR-1106).

---

## 8. SEO Variation Quality Score (Step 12)

```
Quality = 0.30·KeywordCoverage + 0.20·FrontLoading + 0.20·TagDiversity
        + 0.15·Readability + 0.15·CompetitionFit
```

| Component | Computation |
|---|---|
| KeywordCoverage | Σ sales-weighted TF-IDF of matched keywords / Σ of the top-13 available, capped at 1 |
| FrontLoading | Position of the primary keyword in the title: `1 − (charIndex / titleLength)` |
| TagDiversity | `1 − meanPairwiseJaccard(tags)`; penalises 13 near-synonyms |
| Readability | Flesch reading ease of the description mapped to 0–1, plus a stuffing penalty: `−0.3` if any term's repeat ratio > 0.06 |
| CompetitionFit | Prefers keywords with high demand and *moderate* competition: `1 − |competition_percentile − 0.45| / 0.55` |

`CompetitionFit` deliberately does not maximise low competition — a zero-competition keyword is usually a keyword nobody searches.

---

## 9. Learning loop: weight recalibration

### 9.1 Target

```
y = age_normalised_percentile of the listing within its (workspace, product_type) cohort
    measured at outcome_window_days = 90
```
A percentile rather than raw revenue, because it is robust to price differences, seasonality and portfolio composition.

### 9.2 Fit

Ridge regression on the standardised feature matrix from `outcome_features`, with `α` selected by 5-fold cross-validation on a **time-based split** (train on the earliest 70%, test on the most recent 30%). Random splits would leak future information through shared market conditions and are prohibited.

Gradient-boosted trees replace ridge at n ≥ 500, but the linear model remains the reference for interpretability, and any GBDT-derived config must beat the linear one on the holdout to be proposed.

### 9.3 Shrinkage toward the prior

```
λ  = n / (n + k),   k = 150
w_final = λ · w_fitted + (1 − λ) · w_prior
```

At n = 50, `λ = 0.25` — the fitted signal moves the weights by a quarter. At n = 450, `λ = 0.75`. This empirical-Bayes shrinkage is what prevents a run of luck on twelve listings from rewriting the model.

### 9.4 Guards

| Guard | Rule |
|---|---|
| Minimum n | Below 50 outcomes, recalibration is disabled and the UI says so |
| Weight bounds | No weight may move more than 0.15 absolute in a single proposal |
| Sum constraint | Weights within a score renormalise to sum to 1 |
| Sign constraint | A prior-positive weight may not become negative without n ≥ 200 and explicit operator confirmation |
| Back-test gate | A proposal must improve holdout Spearman correlation by ≥ 0.03 to be surfaced at all |
| Activation | Always manual, always reversible (FR-1612, FR-1613) |

### 9.5 Evaluation metrics

| Metric | Use |
|---|---|
| Spearman ρ (predicted vs actual percentile) | Primary ranking quality |
| Brier score on top-quartile classification | Calibration |
| Precision@10 | Practical: of the ten designs the model liked most, how many performed |
| Per-dimension ablation | Which of the five sub-scores is actually carrying predictive power — and which should be down-weighted or removed |

The ablation result is shown to the operator (FR-1616). If, after 200 outcomes, Originality has no predictive power in their niches, they deserve to know that, and the model deserves to reflect it.
