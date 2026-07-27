# 07 — Entity Relationship Diagrams

**Version:** 1.0 · Diagrams are grouped by domain because a single ERD across ~55 tables is unreadable. Cardinalities use crow's-foot notation via Mermaid.

---

## 1. High-level domain map

```mermaid
flowchart TB
    IDENTITY[Identity & Tenancy<br/>workspaces · users · members · audit] 
    CONFIG[Configuration<br/>scoring_configs · fee_models · product_types]
    RESEARCH[Research<br/>niches · sub_niches · runs · run_steps]
    MARKET[Market Data<br/>shops · listings · snapshots · style_profiles]
    ANALYSIS[Analysis<br/>opportunity · success · failure · gaps]
    CREATIVE[Creative<br/>concepts · legal · briefs · artworks]
    COMMERCE[Commerce<br/>blueprints · drafts · variants · seo · pricing]
    PUBLISH[Publishing<br/>integrations · listings · publish_jobs]
    PERF[Performance<br/>performance_snapshots]
    LEARN[Learning<br/>outcome_features · proposals · backtests]
    PLATFORM[Platform<br/>assets · provider_calls · ai_calls]

    IDENTITY --> CONFIG & RESEARCH & MARKET & ANALYSIS & CREATIVE & COMMERCE & PUBLISH
    CONFIG --> ANALYSIS & CREATIVE & COMMERCE
    RESEARCH --> MARKET --> ANALYSIS --> CREATIVE --> COMMERCE --> PUBLISH --> PERF --> LEARN
    LEARN -->|new scoring_config| CONFIG
    PLATFORM -.instrumented by.- RESEARCH & CREATIVE & COMMERCE & PUBLISH
```

---

## 2. Identity, tenancy & configuration

```mermaid
erDiagram
    WORKSPACES ||--o{ WORKSPACE_MEMBERS : has
    USERS      ||--o{ WORKSPACE_MEMBERS : belongs_to
    USERS      ||--o{ SESSIONS : owns
    WORKSPACES ||--o{ SESSIONS : scopes
    WORKSPACES ||--o{ API_KEYS : issues
    WORKSPACES ||--o{ AUDIT_LOG : records
    WORKSPACES ||--o{ SCORING_CONFIGS : versions
    WORKSPACES ||--o{ FEE_MODELS : configures
    WORKSPACES ||--o{ INTEGRATIONS : connects
    INTEGRATIONS ||--|| INTEGRATION_CREDENTIALS : secured_by

    WORKSPACES {
        uuid id PK
        text name
        char3 base_currency
        text plan
        text status
    }
    USERS {
        uuid id PK
        citext email UK
        text password_hash
        bytea totp_secret_enc
    }
    WORKSPACE_MEMBERS {
        uuid id PK
        uuid workspace_id FK
        uuid user_id FK
        enum role
    }
    SCORING_CONFIGS {
        uuid id PK
        uuid workspace_id FK
        int version UK
        text source
        bool is_active
        jsonb weights
    }
    INTEGRATIONS {
        uuid id PK
        uuid workspace_id FK
        text provider
        text status
        text breaker_state
    }
```

**Notes**
- `workspace_members` exists in Phase 1 with exactly one `owner` row. Multi-user is purely additive.
- `scoring_configs` has a partial unique index guaranteeing at most one active config per workspace.
- Credentials are separated from `integrations` so the metadata table can be read freely without touching ciphertext.

---

## 3. Research & orchestration

```mermaid
erDiagram
    WORKSPACES ||--o{ NICHES : owns
    NICHES     ||--o{ SUB_NICHES : contains
    NICHES     ||--o{ RUNS : researched_by
    PRODUCT_TYPES ||--o{ RUNS : targeted_by
    SCORING_CONFIGS ||--o{ RUNS : governs
    RUNS       ||--o{ RUN_STEPS : composed_of
    RUNS       ||--o{ RUN_EVENTS : emits
    RUNS       ||--o{ SUB_NICHES : discovers
    RUNS       ||--o| RUNS : cloned_from

    RUNS {
        uuid id PK
        uuid workspace_id FK
        uuid niche_id FK
        uuid product_type_id FK
        enum requested_style
        enum resolved_style
        enum depth
        enum status
        bigint budget_amount
        bigint spend_amount
        uuid parent_run_id FK
    }
    RUN_STEPS {
        uuid id PK
        uuid run_id FK
        text step_key UK
        int step_order
        enum status
        int attempt
        jsonb output_ref
        bigint cost_amount
    }
    SUB_NICHES {
        uuid id PK
        uuid niche_id FK
        uuid run_id FK
        numeric opportunity_score
        int rank
    }
```

**The `runs → run_steps` relationship is the backbone of the system.** Every unit of work, cost, retry and output pointer hangs off `run_steps`. `output_ref` is a `{table, id}` pointer rather than 15 nullable FK columns, keeping the step table generic while remaining traceable.

---

## 4. Market data

```mermaid
erDiagram
    WORKSPACES        ||--o{ COMPETITOR_SHOPS : observes
    COMPETITOR_SHOPS  ||--o{ COMPETITOR_LISTINGS : sells
    COMPETITOR_LISTINGS ||--o{ LISTING_SNAPSHOTS : captured_as
    COMPETITOR_LISTINGS ||--o{ STYLE_PROFILES : analysed_as
    RUNS              ||--o{ RUN_SHOP_SELECTIONS : selects
    COMPETITOR_SHOPS  ||--o{ RUN_SHOP_SELECTIONS : selected_in
    RUNS              ||--o{ LISTING_SNAPSHOTS : produced_during
    RUNS              ||--o{ KEYWORD_STATS : derives
    ASSETS            ||--o{ STYLE_PROFILES : image_for
    WORKSPACES        ||--o{ MARKET_DATA_IMPORTS : ingests

    COMPETITOR_SHOPS {
        uuid id PK
        bigint etsy_shop_id UK
        int age_months
        int est_monthly_sales
        bigint est_monthly_revenue_amount
        numeric review_velocity_90d
        text data_source
    }
    LISTING_SNAPSHOTS {
        uuid id PK
        uuid listing_id FK
        timestamptz captured_at PK
        bigint price_amount
        int image_count
        int est_monthly_sales
        bool is_bestseller
        int listing_age_days
    }
    STYLE_PROFILES {
        uuid id PK
        uuid listing_id FK
        text image_hash
        enum palette_family
        enum typography
        enum layout
        enum mockup
        vector embedding
    }
```

**Design points**
- `competitor_shops` and `competitor_listings` are *identity* tables (stable facts). All *volatile* facts live in `listing_snapshots`, which is immutable and time-partitioned. This is what makes longitudinal analysis and re-analysis-without-refetch possible.
- `style_profiles` is keyed by `(workspace_id, image_hash, extractor_version)` so the same image is analysed once and reused across runs — the single largest vision-cost saving.
- `run_shop_selections` is the join that records *why* a shop was chosen, not just that it was.

---

## 5. Analysis

```mermaid
erDiagram
    RUNS ||--|| OPPORTUNITY_REPORTS : yields
    RUNS ||--|| SUCCESS_REPORTS : yields
    RUNS ||--|| FAILURE_REPORTS : yields
    RUNS ||--o{ COVERAGE_CELLS : maps
    RUNS ||--o{ MARKET_GAPS : identifies
    SUCCESS_REPORTS ||--o{ SUCCESS_FACTORS : contains
    FAILURE_REPORTS ||--o{ ANTI_FACTORS : contains
    COVERAGE_CELLS  ||--o| MARKET_GAPS : becomes
    SUB_NICHES      ||--o{ COVERAGE_CELLS : dimension_of
    SUB_NICHES      ||--o{ MARKET_GAPS : located_in
    SCORING_CONFIGS ||--o{ OPPORTUNITY_REPORTS : scored_by
    SCORING_CONFIGS ||--o{ MARKET_GAPS : scored_by

    OPPORTUNITY_REPORTS {
        uuid id PK
        uuid run_id FK
        int version UK
        bool is_provisional
        numeric overall_score
        text verdict
        numeric demand_score
        numeric competition_score
        numeric trend_score
        numeric profitability_score
        numeric seasonality_score
    }
    SUCCESS_FACTORS {
        uuid id PK
        uuid success_report_id FK
        text attribute
        text value
        numeric cohort_support_pct
        numeric baseline_support_pct
        numeric lift
        int sample_size
        numeric weight
        bool insufficient_evidence
    }
    MARKET_GAPS {
        uuid id PK
        uuid run_id FK
        uuid coverage_cell_id FK
        numeric gap_score
        numeric demand_index
        int supply_count
        int rank
    }
```

**Versioning:** `opportunity_reports` carries `version` and `is_provisional` so the early provisional score and the final score coexist and can be compared. Reports are never updated in place.

---

## 6. Creative

```mermaid
erDiagram
    RUNS            ||--o{ CONCEPTS : generates
    MARKET_GAPS     ||--o{ CONCEPTS : inspires
    SUB_NICHES      ||--o{ CONCEPTS : targets
    CONCEPTS        ||--o| CONCEPTS : expanded_from
    CONCEPTS        ||--o{ CONCEPT_SCORES : predicted_by
    CONCEPTS        ||--o{ LEGAL_SCREENINGS : screened_by
    LEGAL_SCREENINGS ||--o{ LEGAL_MATCHES : cites
    CONCEPTS        ||--o{ ARTWORK_BRIEFS : briefed_by
    ARTWORK_BRIEFS  ||--o{ ARTWORKS : rendered_as
    ARTWORKS        ||--o{ ARTWORK_ASSETS : has_renditions
    ARTWORKS        ||--o{ ARTWORK_QA_RESULTS : validated_by
    ARTWORKS        ||--o{ LEGAL_SCREENINGS : screened_by
    ARTWORK_ASSETS  }o--|| ASSETS : stored_as
    ARTWORK_ASSETS  ||--o| ARTWORK_ASSETS : derived_from

    CONCEPTS {
        uuid id PK
        uuid run_id FK
        uuid market_gap_id FK
        enum origin
        enum status
        text name
        enum style
        jsonb visual_direction
        vector embedding
        uuid near_duplicate_of FK
    }
    CONCEPT_SCORES {
        uuid id PK
        uuid concept_id FK
        text stage
        numeric market_fit
        numeric originality
        numeric conversion
        numeric competition
        numeric opportunity
        numeric design_success_score
        jsonb contributions
    }
    LEGAL_SCREENINGS {
        uuid id PK
        text subject_type
        uuid subject_id
        enum risk_level
        bool overridden
        uuid overridden_by FK
    }
    ARTWORKS {
        uuid id PK
        uuid concept_id FK
        uuid brief_id FK
        int variant_index
        enum status
        bigint seed
        vector embedding
        numeric originality_score
    }
```

**Two things worth noting:**
1. `concept_scores` is *append-only with a `stage` discriminator*, so the concept-stage prediction and the artwork-stage prediction are both retained. This is what makes the calibration analysis in doc 5 §3.15 possible.
2. `legal_screenings` uses a polymorphic `(subject_type, subject_id)` because concepts, artworks and listing text all need screening with identical semantics. This is the one place polymorphism is accepted, and it is guarded by a check constraint on `subject_type`.

---

## 7. Commerce

```mermaid
erDiagram
    PRODUCT_TYPES     ||--o{ BLUEPRINTS : categorises
    BLUEPRINTS        ||--o{ BLUEPRINT_VARIANTS : offers
    PRINT_PROVIDERS   ||--o{ BLUEPRINT_VARIANTS : fulfils
    RUNS              ||--o{ PRODUCT_RECOMMENDATIONS : suggests
    BLUEPRINTS        ||--o{ PRODUCT_RECOMMENDATIONS : ranked_as
    CONCEPTS          ||--o{ PRODUCT_DRAFTS : becomes
    ARTWORKS          ||--o{ PRODUCT_DRAFTS : illustrates
    BLUEPRINTS        ||--o{ PRODUCT_DRAFTS : built_on
    PRODUCT_DRAFTS    ||--o{ PRODUCT_VARIANTS : configures
    BLUEPRINT_VARIANTS ||--o{ PRODUCT_VARIANTS : instantiated_as
    PRODUCT_DRAFTS    ||--o{ SEO_VARIATIONS : described_by
    PRODUCT_DRAFTS    ||--o{ PRICING_SNAPSHOTS : priced_by
    FEE_MODELS        ||--o{ PRICING_SNAPSHOTS : applies
    PRODUCT_DRAFTS    ||--o{ LISTING_IMAGES : displays

    PRODUCT_DRAFTS {
        uuid id PK
        uuid concept_id FK
        uuid artwork_id FK
        uuid blueprint_id FK
        uuid seo_variation_id FK
        enum status
        bigint retail_price_amount
        numeric margin_pct
        text printify_product_id
        bigint etsy_listing_id
        jsonb checklist
    }
    SEO_VARIATIONS {
        uuid id PK
        uuid product_draft_id FK
        int variation_index
        text axis
        text title
        text[] tags
        jsonb keywords
        numeric quality_score
        bool is_selected
    }
    BLUEPRINT_VARIANTS {
        uuid id PK
        uuid blueprint_id FK
        uuid print_provider_id FK
        int printify_variant_id
        text colour
        text size
        bigint cost_amount
    }
```

**`product_drafts` is the convergence point** of the entire system — concept, artwork, blueprint, pricing and SEO all meet here, and it is the entity the Final Review page renders.

---

## 8. Publishing & performance

```mermaid
erDiagram
    PRODUCT_DRAFTS ||--o| LISTINGS : published_as
    PRODUCT_DRAFTS ||--o{ PUBLISH_JOBS : executed_by
    LISTINGS       ||--o{ LISTING_IMAGES : shows
    LISTINGS       ||--o{ PERFORMANCE_SNAPSHOTS : measured_by
    LISTINGS       ||--o| OUTCOME_FEATURES : featurised_as
    CONCEPTS       ||--o{ LISTINGS : originated
    RUNS           ||--o{ LISTINGS : traced_to
    INTEGRATIONS   ||--o{ PUBLISH_JOBS : authorises

    LISTINGS {
        uuid id PK
        uuid product_draft_id FK
        uuid concept_id FK
        uuid run_id FK
        bigint etsy_listing_id UK
        enum state
        timestamptz published_at
        timestamptz first_sale_at
        numeric predicted_score
    }
    PUBLISH_JOBS {
        uuid id PK
        uuid product_draft_id FK
        text operation
        text status
        text idempotency_key UK
        int attempt
    }
    PERFORMANCE_SNAPSHOTS {
        uuid id PK
        uuid listing_id FK
        timestamptz captured_at PK
        int views
        int favourites
        int orders
        bigint revenue_amount
    }
```

`publish_jobs` decomposes publishing into seven independently idempotent operations, which is what makes partial-failure recovery (FR-1410) possible without duplicate listings.

---

## 9. Learning loop

```mermaid
erDiagram
    LISTINGS          ||--|| OUTCOME_FEATURES : featurised_as
    PERFORMANCE_SNAPSHOTS }o--|| LISTINGS : informs
    OUTCOME_FEATURES  }o--o{ MODEL_PROPOSALS : trains
    SCORING_CONFIGS   ||--o{ MODEL_PROPOSALS : proposed_from
    MODEL_PROPOSALS   ||--|| SCORING_CONFIGS : proposes
    MODEL_PROPOSALS   ||--o{ MODEL_BACKTESTS : validated_by
    CONCEPT_SCORES    ||--o{ PREDICTION_RECORDS : evaluated_as
    LISTINGS          ||--o{ PREDICTION_RECORDS : realises

    OUTCOME_FEATURES {
        uuid id PK
        uuid listing_id FK
        jsonb features
        text feature_set_version
        int outcome_window_days
        numeric age_normalised_percentile
        bool is_success
    }
    MODEL_PROPOSALS {
        uuid id PK
        uuid base_config_id FK
        uuid proposed_config_id FK
        text method
        int n_outcomes
        jsonb weight_diff
        text status
    }
```

**The cycle:** `listings → performance_snapshots → outcome_features → model_proposals → scoring_configs → (next run's) concept_scores`. Every arrow is an explicit table, so the loop is inspectable rather than hidden inside a training script.

---

## 10. Full lineage chain (the traceability guarantee)

Any published listing can be traced backwards through a single chain of foreign keys:

```mermaid
flowchart LR
    L[listings] --> PD[product_drafts]
    PD --> AW[artworks] --> AB[artwork_briefs] --> C[concepts]
    C --> MG[market_gaps] --> CC[coverage_cells] --> SN[sub_niches]
    C --> SF[success_factors] --> SR[success_reports]
    SR --> R[runs] --> N[niches]
    PD --> SEO[seo_variations]
    PD --> BP[blueprints] --> BV[blueprint_variants]
    C --> LS[legal_screenings] --> LM[legal_matches]
    L --> PS[performance_snapshots] --> OF[outcome_features]
```

**Requirement:** the Listing Detail page (doc 5 §3.14) renders this chain as clickable breadcrumbs. If any link cannot be resolved, that is a data-integrity defect caught by the nightly consistency job (NFR §2.2).

---

## 11. Cardinality reference

| Relationship | Cardinality | Notes |
|---|---|---|
| workspace → everything | 1:N | Tenancy root |
| niche → runs | 1:N | Re-researchable over time |
| run → run_steps | 1:N (≈14) | Fixed by the DAG for a given depth |
| run → competitor_shops | N:M via `run_shop_selections` | Shops are shared across runs |
| competitor_listing → listing_snapshots | 1:N | Immutable time series |
| competitor_listing → style_profiles | 1:N | One per extractor version |
| run → success_report | 1:1 | Regeneration creates a new run or a new report version |
| success_report → success_factors | 1:N (10–40) | Ranked |
| run → concepts | 1:N (20 base + expansions) | |
| concept → concept_scores | 1:N | One per stage per config version |
| concept → legal_screenings | 1:N | Re-screened on edit |
| concept → artwork_briefs | 1:N | Versioned |
| artwork_brief → artworks | 1:N (1–8) | Variants |
| artwork → artwork_assets | 1:N (4–7) | Rendition tree |
| concept + artwork → product_draft | 1:1 typical, 1:N possible | Same artwork can back several products |
| product_draft → seo_variations | 1:N (10) | Exactly one selected |
| product_draft → listing | 1:0..1 | Only after publish |
| listing → performance_snapshots | 1:N (daily) | |
| listing → outcome_features | 1:N | One per window × feature-set version |

---

## 12. Referential-integrity rules

| Rule | Enforcement |
|---|---|
| Deleting a workspace cascades everything | `ON DELETE CASCADE` from `workspaces` |
| Deleting a run must NOT delete shared market data | `runs → competitor_shops` is via join table; snapshots keep `run_id` nullable-on-delete |
| Deleting a concept cascades briefs/artworks/screenings | `ON DELETE CASCADE` |
| A published listing can never be hard-deleted | Soft delete only; DB trigger rejects `DELETE` on `listings` with `published_at IS NOT NULL` |
| `product_drafts.seo_variation_id` must reference a variation belonging to the same draft | Deferred check via trigger |
| `artworks.concept_id` must match `artwork_briefs.concept_id` | Composite FK on `(concept_id, brief_id)` |
| An artwork cannot be attached to a draft unless its latest QA result is `pass` | Enforced in the service layer + a nightly consistency assertion |
| A concept cannot have artwork unless its latest screening passed | Enforced in the service layer (FR-901) + nightly assertion |
