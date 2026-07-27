# 06 — Database Schema Design

**Version:** 1.0 · PostgreSQL 16 · Extensions: `pgcrypto`, `pg_trgm`, `vector` (pgvector), `btree_gin`, `pg_stat_statements`

---

## 1. Conventions

| Rule | Detail |
|---|---|
| Primary keys | `uuid` v7 (time-ordered) generated in the application; column name `id` |
| Tenancy | **Every** domain table has `workspace_id uuid NOT NULL REFERENCES workspaces(id)`. Composite indexes always lead with `workspace_id`. |
| Timestamps | `timestamptz`, UTC. `created_at`, `updated_at` on all mutable tables. `deleted_at` where soft delete applies. |
| Money | `bigint` minor units + `char(3)` ISO-4217 currency, named `*_amount` / `*_currency`. Never `float`/`numeric` for money. |
| Percentages/ratios | `numeric(8,4)` for ratios, `numeric(5,2)` for 0–100 scores |
| Enums | Native Postgres enums for closed, stable vocabularies; lookup tables where the operator may extend |
| JSON | `jsonb` for provider payloads, feature vectors and flexible metadata; never for anything queried in a hot path unless GIN-indexed |
| Naming | snake_case; tables plural; FK columns `<singular>_id` |
| Soft delete | `deleted_at timestamptz NULL`; all reads filter it; hard purge job after 30 days |
| Immutability | Snapshot and ledger tables have no `updated_at`; corrections are new rows |
| Partitioning | `listing_snapshots`, `performance_snapshots`, `provider_calls`, `ai_calls`, `events` are declared as `PARTITION BY RANGE (created_at)` from day one (monthly partitions) |
| Audit | `audit_log` is append-only, no updates or deletes permitted (enforced by a rule + revoked grants) |

---

## 2. Schema domains

```
identity      workspaces, users, workspace_members, sessions, api_keys, audit_log
config        scoring_configs, product_types, style_presets, workspace_settings, fee_models
research      niches, sub_niches, runs, run_steps, run_events, run_budgets
market        competitor_shops, competitor_listings, listing_snapshots, style_profiles,
              keyword_stats, market_data_imports
analysis      opportunity_reports, success_reports, success_factors, failure_reports,
              anti_factors, market_gaps, coverage_cells, correlations
creative      concepts, concept_scores, legal_screenings, legal_matches,
              artwork_briefs, artworks, artwork_assets, artwork_qa_results
commerce      blueprints, print_providers, blueprint_variants, product_drafts,
              product_variants, seo_variations, pricing_snapshots
publishing    integrations, integration_credentials, listings, listing_images,
              publish_jobs, performance_snapshots
learning      outcome_features, model_proposals, model_backtests, prediction_records
platform      provider_calls, ai_calls, assets, jobs_dlq, feature_flags, notifications
billing       (SaaS phase) plans, subscriptions, usage_events, invoices  — see doc 21
```

---

## 3. DDL

### 3.1 Identity & tenancy

```sql
CREATE TABLE workspaces (
  id              uuid PRIMARY KEY,
  name            text NOT NULL,
  slug            citext NOT NULL UNIQUE,
  base_currency   char(3) NOT NULL DEFAULT 'GBP',
  country_code    char(2) NOT NULL DEFAULT 'GB',
  timezone        text NOT NULL DEFAULT 'Europe/London',
  plan            text NOT NULL DEFAULT 'personal',       -- SaaS-ready
  status          text NOT NULL DEFAULT 'active',          -- active|suspended|closed
  settings        jsonb NOT NULL DEFAULT '{}'::jsonb,
  created_at      timestamptz NOT NULL DEFAULT now(),
  updated_at      timestamptz NOT NULL DEFAULT now(),
  deleted_at      timestamptz
);

CREATE TABLE users (
  id                 uuid PRIMARY KEY,
  email              citext NOT NULL UNIQUE,
  password_hash      text NOT NULL,                        -- Argon2id
  totp_secret_enc    bytea,                                -- envelope-encrypted
  totp_enabled       boolean NOT NULL DEFAULT false,
  recovery_codes_enc bytea,
  display_name       text,
  last_login_at      timestamptz,
  failed_logins      int NOT NULL DEFAULT 0,
  locked_until       timestamptz,
  created_at         timestamptz NOT NULL DEFAULT now(),
  updated_at         timestamptz NOT NULL DEFAULT now(),
  deleted_at         timestamptz
);

-- Present from day one even with a single member; the SaaS phase only adds rows.
CREATE TYPE workspace_role AS ENUM ('owner','admin','member','viewer');

CREATE TABLE workspace_members (
  id           uuid PRIMARY KEY,
  workspace_id uuid NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
  user_id      uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  role         workspace_role NOT NULL DEFAULT 'owner',
  invited_by   uuid REFERENCES users(id),
  joined_at    timestamptz NOT NULL DEFAULT now(),
  UNIQUE (workspace_id, user_id)
);
CREATE INDEX ON workspace_members (user_id);

CREATE TABLE sessions (
  id             uuid PRIMARY KEY,
  user_id        uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  workspace_id   uuid NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
  token_hash     text NOT NULL UNIQUE,
  ip_hash        text,
  user_agent     text,
  expires_at     timestamptz NOT NULL,
  revoked_at     timestamptz,
  created_at     timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX ON sessions (user_id, expires_at);

-- SaaS phase; defined now so the shape is fixed.
CREATE TABLE api_keys (
  id             uuid PRIMARY KEY,
  workspace_id   uuid NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
  name           text NOT NULL,
  key_prefix     text NOT NULL,
  key_hash       text NOT NULL UNIQUE,
  scopes         text[] NOT NULL DEFAULT '{}',
  last_used_at   timestamptz,
  expires_at     timestamptz,
  revoked_at     timestamptz,
  created_by     uuid REFERENCES users(id),
  created_at     timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE audit_log (
  id             uuid PRIMARY KEY,
  workspace_id   uuid NOT NULL,
  actor_user_id  uuid,
  actor_type     text NOT NULL DEFAULT 'user',   -- user|system|api_key
  action         text NOT NULL,                  -- e.g. listing.publish, legal.override
  entity_type    text NOT NULL,
  entity_id      uuid,
  before_state   jsonb,
  after_state    jsonb,
  justification  text,
  ip_hash        text,
  correlation_id text,
  created_at     timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX ON audit_log (workspace_id, created_at DESC);
CREATE INDEX ON audit_log (entity_type, entity_id);
```

### 3.2 Configuration

```sql
CREATE TABLE scoring_configs (
  id             uuid PRIMARY KEY,
  workspace_id   uuid NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
  version        int NOT NULL,
  name           text NOT NULL,
  source         text NOT NULL,        -- expert_prior|fitted|manual
  is_active      boolean NOT NULL DEFAULT false,
  weights        jsonb NOT NULL,       -- full weight tree; see Appendix A
  normalisation  jsonb NOT NULL,       -- per-feature scaling params
  thresholds     jsonb NOT NULL,       -- band cutoffs, min sample sizes, demand floor
  fitted_from    jsonb,                -- {n_outcomes, date_range, method, r2, brier}
  activated_at   timestamptz,
  activated_by   uuid REFERENCES users(id),
  notes          text,
  created_at     timestamptz NOT NULL DEFAULT now(),
  UNIQUE (workspace_id, version)
);
CREATE UNIQUE INDEX one_active_config_per_workspace
  ON scoring_configs (workspace_id) WHERE is_active;

CREATE TABLE product_types (
  id            uuid PRIMARY KEY,
  code          text NOT NULL UNIQUE,      -- tshirt|sweatshirt|hoodie|mug|poster|tote
  name          text NOT NULL,
  category      text NOT NULL,             -- apparel|drinkware|wall_art|accessory
  default_print_area jsonb,                -- {width_in, height_in, dpi}
  etsy_taxonomy_id int,
  active        boolean NOT NULL DEFAULT true
);

CREATE TYPE design_style AS ENUM
  ('vintage','typography','hand_drawn','illustration','humour','modern','auto');

CREATE TABLE fee_models (
  id                       uuid PRIMARY KEY,
  workspace_id             uuid NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
  name                     text NOT NULL,
  listing_fee_amount       bigint NOT NULL DEFAULT 20,     -- £0.20 equivalent
  listing_fee_currency     char(3) NOT NULL DEFAULT 'GBP',
  transaction_pct          numeric(6,4) NOT NULL DEFAULT 0.0650,
  payment_pct              numeric(6,4) NOT NULL DEFAULT 0.0400,
  payment_fixed_amount     bigint NOT NULL DEFAULT 20,
  offsite_ads_pct          numeric(6,4) NOT NULL DEFAULT 0.1500,
  offsite_ads_applies      boolean NOT NULL DEFAULT true,
  regulatory_pct           numeric(6,4) NOT NULL DEFAULT 0.0032,
  vat_pct                  numeric(6,4) NOT NULL DEFAULT 0.2000,
  vat_applies              boolean NOT NULL DEFAULT false,
  margin_floor_pct         numeric(6,4) NOT NULL DEFAULT 0.4000,
  is_default               boolean NOT NULL DEFAULT true,
  created_at               timestamptz NOT NULL DEFAULT now(),
  updated_at               timestamptz NOT NULL DEFAULT now()
);
```

### 3.3 Research

```sql
CREATE TABLE niches (
  id                uuid PRIMARY KEY,
  workspace_id      uuid NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
  name              text NOT NULL,
  normalised_name   text NOT NULL,
  description       text,
  status            text NOT NULL DEFAULT 'active',  -- active|rejected|exhausted
  rejection_reason  text,
  best_opportunity_score numeric(5,2),
  last_researched_at timestamptz,
  created_at        timestamptz NOT NULL DEFAULT now(),
  updated_at        timestamptz NOT NULL DEFAULT now(),
  deleted_at        timestamptz,
  UNIQUE (workspace_id, normalised_name)
);
CREATE INDEX niches_trgm ON niches USING gin (normalised_name gin_trgm_ops);

CREATE TABLE sub_niches (
  id                 uuid PRIMARY KEY,
  workspace_id       uuid NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
  niche_id           uuid NOT NULL REFERENCES niches(id) ON DELETE CASCADE,
  run_id             uuid,                       -- FK added after runs
  name               text NOT NULL,
  normalised_name    text NOT NULL,
  description        text,
  rationale          text,
  demand_index       numeric(6,2),
  competition_index  numeric(6,2),
  opportunity_score  numeric(5,2),
  rank               int,
  example_terms      text[] NOT NULL DEFAULT '{}',
  evidence_sources   text[] NOT NULL DEFAULT '{}',   -- llm|cooccurrence|taxonomy
  listing_count_observed int NOT NULL DEFAULT 0,
  scoring_config_version int,
  created_at         timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX ON sub_niches (workspace_id, niche_id, rank);

CREATE TYPE run_status AS ENUM
  ('queued','running','paused_budget','awaiting_selection','partially_failed',
   'failed','completed','cancelled');
CREATE TYPE run_depth AS ENUM ('quick','standard','deep');

CREATE TABLE runs (
  id                    uuid PRIMARY KEY,
  workspace_id          uuid NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
  niche_id              uuid NOT NULL REFERENCES niches(id),
  product_type_id       uuid NOT NULL REFERENCES product_types(id),
  requested_style       design_style NOT NULL DEFAULT 'auto',
  resolved_style        design_style,
  resolved_style_reason text,
  depth                 run_depth NOT NULL DEFAULT 'standard',
  status                run_status NOT NULL DEFAULT 'queued',
  seed_keywords         text[] NOT NULL DEFAULT '{}',
  excluded_terms        text[] NOT NULL DEFAULT '{}',
  degraded              boolean NOT NULL DEFAULT false,
  degradation_reasons   text[] NOT NULL DEFAULT '{}',
  scoring_config_id     uuid NOT NULL REFERENCES scoring_configs(id),
  budget_amount         bigint NOT NULL,
  budget_currency       char(3) NOT NULL DEFAULT 'GBP',
  spend_amount          bigint NOT NULL DEFAULT 0,
  idempotency_key       text,
  parent_run_id         uuid REFERENCES runs(id),      -- clone/refine lineage
  started_at            timestamptz,
  finished_at           timestamptz,
  cancelled_at          timestamptz,
  error                 jsonb,
  created_by            uuid REFERENCES users(id),
  created_at            timestamptz NOT NULL DEFAULT now(),
  updated_at            timestamptz NOT NULL DEFAULT now(),
  deleted_at            timestamptz
);
CREATE INDEX ON runs (workspace_id, status, created_at DESC);
CREATE INDEX ON runs (workspace_id, niche_id, created_at DESC);
CREATE UNIQUE INDEX runs_idem ON runs (workspace_id, idempotency_key)
  WHERE idempotency_key IS NOT NULL;

ALTER TABLE sub_niches ADD CONSTRAINT sub_niches_run_fk
  FOREIGN KEY (run_id) REFERENCES runs(id) ON DELETE SET NULL;

CREATE TYPE step_status AS ENUM
  ('pending','running','succeeded','failed','skipped','blocked_external','cancelled');

CREATE TABLE run_steps (
  id              uuid PRIMARY KEY,
  workspace_id    uuid NOT NULL,
  run_id          uuid NOT NULL REFERENCES runs(id) ON DELETE CASCADE,
  step_key        text NOT NULL,        -- opportunity|competitors|style_extract|success|...
  step_order      int NOT NULL,
  status          step_status NOT NULL DEFAULT 'pending',
  attempt         int NOT NULL DEFAULT 0,
  max_attempts    int NOT NULL DEFAULT 3,
  depends_on      text[] NOT NULL DEFAULT '{}',
  input_hash      text,
  output_ref      jsonb,                -- {table, id} pointers to the produced entity
  cost_amount     bigint NOT NULL DEFAULT 0,
  cost_currency   char(3) NOT NULL DEFAULT 'GBP',
  tokens_in       bigint NOT NULL DEFAULT 0,
  tokens_out      bigint NOT NULL DEFAULT 0,
  started_at      timestamptz,
  finished_at     timestamptz,
  duration_ms     int,
  error           jsonb,
  created_at      timestamptz NOT NULL DEFAULT now(),
  updated_at      timestamptz NOT NULL DEFAULT now(),
  UNIQUE (run_id, step_key)
);
CREATE INDEX ON run_steps (run_id, step_order);
CREATE INDEX ON run_steps (status) WHERE status IN ('running','pending');

CREATE TABLE run_events (           -- progress stream, partitioned monthly
  id            uuid NOT NULL,
  workspace_id  uuid NOT NULL,
  run_id        uuid NOT NULL,
  step_key      text,
  level         text NOT NULL DEFAULT 'info',
  event         text NOT NULL,
  message       text,
  data          jsonb,
  created_at    timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);
CREATE INDEX ON run_events (run_id, created_at);
```

### 3.4 Market data

```sql
CREATE TABLE competitor_shops (
  id                     uuid PRIMARY KEY,
  workspace_id           uuid NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
  etsy_shop_id           bigint,
  shop_name              text NOT NULL,
  shop_url               text,
  opened_at              date,
  age_months             int,
  location               text,
  total_sales            int,
  total_reviews          int,
  average_rating         numeric(3,2),
  active_listing_count   int,
  est_monthly_sales      int,
  est_monthly_revenue_amount   bigint,
  est_monthly_revenue_currency char(3),
  review_velocity_90d    numeric(8,2),
  data_source            text NOT NULL,             -- everbee_csv|everbee_session|etsy_api|manual
  data_confidence        text NOT NULL DEFAULT 'medium',
  first_seen_at          timestamptz NOT NULL DEFAULT now(),
  last_seen_at           timestamptz NOT NULL DEFAULT now(),
  created_at             timestamptz NOT NULL DEFAULT now(),
  updated_at             timestamptz NOT NULL DEFAULT now(),
  UNIQUE (workspace_id, etsy_shop_id)
);
CREATE INDEX ON competitor_shops (workspace_id, est_monthly_revenue_amount DESC NULLS LAST);

CREATE TABLE run_shop_selections (
  id                 uuid PRIMARY KEY,
  workspace_id       uuid NOT NULL,
  run_id             uuid NOT NULL REFERENCES runs(id) ON DELETE CASCADE,
  shop_id            uuid NOT NULL REFERENCES competitor_shops(id),
  selection_score    numeric(8,3) NOT NULL,
  rank               int NOT NULL,
  criteria_passed    jsonb NOT NULL,      -- {sales:true, reviews:true, age_bonus:1.10, ...}
  listings_collected int NOT NULL DEFAULT 0,
  truncated          boolean NOT NULL DEFAULT false,
  created_at         timestamptz NOT NULL DEFAULT now(),
  UNIQUE (run_id, shop_id)
);

CREATE TABLE competitor_listings (
  id                 uuid PRIMARY KEY,
  workspace_id       uuid NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
  shop_id            uuid NOT NULL REFERENCES competitor_shops(id) ON DELETE CASCADE,
  etsy_listing_id    bigint,
  title              text NOT NULL,
  url                text,
  product_type_id    uuid REFERENCES product_types(id),
  listed_at          date,
  first_seen_at      timestamptz NOT NULL DEFAULT now(),
  last_seen_at       timestamptz NOT NULL DEFAULT now(),
  is_active          boolean NOT NULL DEFAULT true,
  title_tsv          tsvector GENERATED ALWAYS AS (to_tsvector('english', title)) STORED,
  created_at         timestamptz NOT NULL DEFAULT now(),
  updated_at         timestamptz NOT NULL DEFAULT now(),
  UNIQUE (workspace_id, etsy_listing_id)
);
CREATE INDEX ON competitor_listings USING gin (title_tsv);
CREATE INDEX ON competitor_listings (workspace_id, shop_id);

-- Immutable observations. Partitioned monthly.
CREATE TABLE listing_snapshots (
  id                      uuid NOT NULL,
  workspace_id            uuid NOT NULL,
  run_id                  uuid,
  listing_id              uuid NOT NULL,
  captured_at             timestamptz NOT NULL DEFAULT now(),
  price_amount            bigint,
  price_currency          char(3),
  shipping_amount         bigint,
  free_shipping           boolean,
  image_count             int,
  primary_image_asset_id  uuid,
  description             text,
  tags                    text[] NOT NULL DEFAULT '{}',
  materials               text[] NOT NULL DEFAULT '{}',
  review_count            int,
  average_rating          numeric(3,2),
  review_velocity_90d     numeric(8,2),
  favourites              int,
  views                   int,
  est_monthly_sales       int,
  est_total_sales         int,
  est_monthly_revenue_amount bigint,
  est_revenue_currency    char(3),
  is_bestseller           boolean,
  has_personalisation     boolean,
  listing_age_days        int,
  shop_age_months         int,
  section                 text,
  data_source             text NOT NULL,
  data_confidence         text NOT NULL DEFAULT 'medium',
  raw                     jsonb,
  PRIMARY KEY (id, captured_at)
) PARTITION BY RANGE (captured_at);
CREATE INDEX ON listing_snapshots (workspace_id, listing_id, captured_at DESC);
CREATE INDEX ON listing_snapshots (run_id, est_monthly_sales DESC);

CREATE TYPE palette_family AS ENUM
  ('muted_earth','muted_green','high_contrast_mono','pastel','neon','vintage_washed',
   'jewel','monochrome_dark','natural_neutral','warm_retro','cool_modern','other');
CREATE TYPE typography_style AS ENUM
  ('vintage_serif','condensed_sans','script','handwritten','slab','display_bold',
   'retro_groovy','minimal_sans','distressed','none');
CREATE TYPE layout_archetype AS ENUM
  ('centred_stack','badge_circle','arched_text','left_aligned_block','illustration_only',
   'text_over_illustration','split','border_frame','repeat_pattern','other');
CREATE TYPE mockup_style AS ENUM
  ('flat_lay','model_lifestyle','ghost_mannequin','hanging','folded','studio_plain',
   'graphic_only','in_situ_scene','other');

CREATE TABLE style_profiles (
  id                 uuid PRIMARY KEY,
  workspace_id       uuid NOT NULL,
  listing_id         uuid NOT NULL REFERENCES competitor_listings(id) ON DELETE CASCADE,
  snapshot_id        uuid,
  image_hash         text NOT NULL,
  palette            jsonb NOT NULL,        -- [{hex, proportion, lab:[l,a,b]}]
  palette_family     palette_family,
  dominant_hex       text,
  typography         typography_style,
  typography_confidence numeric(4,3),
  layout             layout_archetype,
  mockup             mockup_style,
  subject_tags       text[] NOT NULL DEFAULT '{}',
  humour_type        text,                  -- pun|sarcasm|wholesome|none
  has_text           boolean,
  text_length_band   text,                  -- none|short|medium|long
  garment_colour     text,
  embedding          vector(512),
  extractor_version  text NOT NULL,
  extracted_at       timestamptz NOT NULL DEFAULT now(),
  UNIQUE (workspace_id, image_hash, extractor_version)
);
CREATE INDEX ON style_profiles (workspace_id, palette_family);
CREATE INDEX style_profiles_embedding ON style_profiles
  USING hnsw (embedding vector_cosine_ops);

CREATE TABLE keyword_stats (
  id                 uuid PRIMARY KEY,
  workspace_id       uuid NOT NULL,
  run_id             uuid NOT NULL REFERENCES runs(id) ON DELETE CASCADE,
  term               text NOT NULL,
  term_type          text NOT NULL,        -- tag|title_ngram|description_ngram
  frequency          int NOT NULL,
  doc_frequency      int NOT NULL,
  tfidf              numeric(10,6),
  sales_weighted_score numeric(12,4),
  used_by_top_decile_pct numeric(5,2),
  avg_price_amount   bigint,
  competition_index  numeric(6,2),
  created_at         timestamptz NOT NULL DEFAULT now(),
  UNIQUE (run_id, term, term_type)
);
CREATE INDEX ON keyword_stats (run_id, sales_weighted_score DESC);

CREATE TABLE market_data_imports (
  id             uuid PRIMARY KEY,
  workspace_id   uuid NOT NULL,
  source         text NOT NULL,            -- everbee_csv|manual_csv
  filename       text,
  asset_id       uuid,
  row_count      int,
  accepted_count int,
  rejected_count int,
  mapping        jsonb,                    -- column mapping used
  errors         jsonb,
  imported_by    uuid REFERENCES users(id),
  created_at     timestamptz NOT NULL DEFAULT now()
);
```

### 3.5 Analysis

```sql
CREATE TABLE opportunity_reports (
  id                     uuid PRIMARY KEY,
  workspace_id           uuid NOT NULL,
  run_id                 uuid NOT NULL REFERENCES runs(id) ON DELETE CASCADE,
  niche_id               uuid NOT NULL REFERENCES niches(id),
  version                int NOT NULL DEFAULT 1,
  is_provisional         boolean NOT NULL DEFAULT false,
  overall_score          numeric(5,2) NOT NULL,
  verdict                text NOT NULL,        -- avoid|marginal|good|strong|exceptional
  demand_score           numeric(5,2) NOT NULL,
  competition_score      numeric(5,2) NOT NULL,
  trend_score            numeric(5,2) NOT NULL,
  profitability_score    numeric(5,2) NOT NULL,
  seasonality_score      numeric(5,2) NOT NULL,
  sub_score_details      jsonb NOT NULL,       -- per sub-score: features, method, confidence, explanation
  seasonality_index      numeric(6,2)[] NOT NULL,  -- 12 values
  peak_months            int[] NOT NULL DEFAULT '{}',
  trough_months          int[] NOT NULL DEFAULT '{}',
  seasonality_strength   numeric(6,4),
  executive_summary      text,
  percentile_in_workspace numeric(5,2),
  degraded               boolean NOT NULL DEFAULT false,
  scoring_config_id      uuid NOT NULL REFERENCES scoring_configs(id),
  created_at             timestamptz NOT NULL DEFAULT now(),
  UNIQUE (run_id, version)
);

CREATE TABLE success_reports (
  id                 uuid PRIMARY KEY,
  workspace_id       uuid NOT NULL,
  run_id             uuid NOT NULL REFERENCES runs(id) ON DELETE CASCADE,
  cohort_definition  jsonb NOT NULL,   -- {metric, percentile, n, total, min_age_days}
  cohort_size        int NOT NULL,
  population_size    int NOT NULL,
  synthesis          jsonb NOT NULL,   -- "median winning listing" spec sheet
  correlations       jsonb,            -- Spearman matrix + multicollinearity flags
  interactions       jsonb,
  resolved_style     design_style,
  resolved_style_evidence jsonb,
  scoring_config_id  uuid NOT NULL REFERENCES scoring_configs(id),
  created_at         timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE success_factors (
  id                 uuid PRIMARY KEY,
  workspace_id       uuid NOT NULL,
  success_report_id  uuid NOT NULL REFERENCES success_reports(id) ON DELETE CASCADE,
  attribute_group    text NOT NULL,    -- colour|typography|layout|pricing|presentation|seo|format
  attribute          text NOT NULL,    -- palette_family
  value              text,             -- muted_green
  numeric_band       jsonb,            -- {min, max} for numeric attributes
  statement          text NOT NULL,
  cohort_support_pct numeric(5,2) NOT NULL,
  baseline_support_pct numeric(5,2) NOT NULL,
  lift               numeric(8,3) NOT NULL,
  sample_size        int NOT NULL,
  p_value            numeric(10,8),
  effect_size        numeric(8,4),
  confidence         text NOT NULL,    -- low|medium|high
  weight             numeric(5,4) NOT NULL,
  insufficient_evidence boolean NOT NULL DEFAULT false,
  supporting_listing_ids uuid[] NOT NULL DEFAULT '{}',
  rank               int,
  created_at         timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX ON success_factors (success_report_id, rank);

CREATE TABLE failure_reports (
  id                 uuid PRIMARY KEY,
  workspace_id       uuid NOT NULL,
  run_id             uuid NOT NULL REFERENCES runs(id) ON DELETE CASCADE,
  cohort_definition  jsonb NOT NULL,
  cohort_size        int NOT NULL,
  population_size    int NOT NULL,
  crowded_losers     jsonb,
  scoring_config_id  uuid NOT NULL REFERENCES scoring_configs(id),
  created_at         timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE anti_factors (
  id                 uuid PRIMARY KEY,
  workspace_id       uuid NOT NULL,
  failure_report_id  uuid NOT NULL REFERENCES failure_reports(id) ON DELETE CASCADE,
  attribute_group    text NOT NULL,
  attribute          text NOT NULL,
  value              text,
  numeric_band       jsonb,
  statement          text NOT NULL,
  cohort_support_pct numeric(5,2) NOT NULL,
  baseline_support_pct numeric(5,2) NOT NULL,
  lift               numeric(8,3) NOT NULL,
  sample_size        int NOT NULL,
  p_value            numeric(10,8),
  confidence         text NOT NULL,
  penalty_weight     numeric(5,4) NOT NULL,
  causality          text NOT NULL DEFAULT 'correlational_only',  -- causal_plausible|correlational_only
  ambiguous          boolean NOT NULL DEFAULT false,  -- also appears as a success factor
  rank               int,
  created_at         timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE coverage_cells (
  id             uuid PRIMARY KEY,
  workspace_id   uuid NOT NULL,
  run_id         uuid NOT NULL REFERENCES runs(id) ON DELETE CASCADE,
  sub_niche_id   uuid REFERENCES sub_niches(id),
  design_angle   text NOT NULL,
  style          design_style NOT NULL,
  supply_count   int NOT NULL DEFAULT 0,
  demand_index   numeric(6,2) NOT NULL,
  monetisability numeric(6,2),
  avg_price_amount bigint,
  created_at     timestamptz NOT NULL DEFAULT now(),
  UNIQUE (run_id, sub_niche_id, design_angle, style)
);

CREATE TABLE market_gaps (
  id                  uuid PRIMARY KEY,
  workspace_id        uuid NOT NULL,
  run_id              uuid NOT NULL REFERENCES runs(id) ON DELETE CASCADE,
  coverage_cell_id    uuid REFERENCES coverage_cells(id),
  sub_niche_id        uuid REFERENCES sub_niches(id),
  title               text NOT NULL,
  design_angle        text NOT NULL,
  style               design_style,
  gap_score           numeric(5,2) NOT NULL,
  demand_index        numeric(6,2) NOT NULL,
  supply_count        int NOT NULL,
  monetisability      numeric(6,2),
  feasibility         numeric(6,2),
  explanation         text NOT NULL,
  demand_evidence     jsonb NOT NULL,
  supply_evidence     jsonb NOT NULL,
  suggested_angles    text[] NOT NULL DEFAULT '{}',
  caution_flags       text[] NOT NULL DEFAULT '{}',   -- trademark_heavy|seasonal_dead|unprintable
  rank                int NOT NULL,
  scoring_config_id   uuid NOT NULL REFERENCES scoring_configs(id),
  created_at          timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX ON market_gaps (run_id, rank);
```

### 3.6 Creative

```sql
CREATE TYPE concept_origin AS ENUM ('success','gap','manual','expansion');
CREATE TYPE concept_status AS ENUM
  ('generated','selected','screening','cleared','blocked','artwork_pending',
   'artwork_ready','archived','rejected');

CREATE TABLE concepts (
  id                  uuid PRIMARY KEY,
  workspace_id        uuid NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
  run_id              uuid REFERENCES runs(id) ON DELETE SET NULL,
  niche_id            uuid REFERENCES niches(id),
  sub_niche_id        uuid REFERENCES sub_niches(id),
  market_gap_id       uuid REFERENCES market_gaps(id),
  parent_concept_id   uuid REFERENCES concepts(id),
  origin              concept_origin NOT NULL,
  status              concept_status NOT NULL DEFAULT 'generated',
  name                text NOT NULL,
  description         text NOT NULL,
  target_audience     text NOT NULL,
  design_angle        text,
  style               design_style NOT NULL,
  visual_direction    jsonb NOT NULL,    -- {palette_family, hexes[], typography, layout}
  text_content        text,
  reasoning           text NOT NULL,
  cited_factor_ids    uuid[] NOT NULL DEFAULT '{}',
  cited_gap_ids       uuid[] NOT NULL DEFAULT '{}',
  embedding           vector(1024),
  near_duplicate_of   uuid REFERENCES concepts(id),
  similarity_score    numeric(5,4),
  prompt_version      text NOT NULL,
  selected_at         timestamptz,
  created_at          timestamptz NOT NULL DEFAULT now(),
  updated_at          timestamptz NOT NULL DEFAULT now(),
  deleted_at          timestamptz
);
CREATE INDEX ON concepts (workspace_id, run_id, origin);
CREATE INDEX ON concepts (workspace_id, status);
CREATE INDEX concepts_embedding ON concepts USING hnsw (embedding vector_cosine_ops);

CREATE TABLE concept_scores (
  id                  uuid PRIMARY KEY,
  workspace_id        uuid NOT NULL,
  concept_id          uuid NOT NULL REFERENCES concepts(id) ON DELETE CASCADE,
  artwork_id          uuid,                    -- null = scored at concept stage
  stage               text NOT NULL,           -- concept|artwork
  market_fit          numeric(5,2) NOT NULL,
  originality         numeric(5,2) NOT NULL,
  conversion          numeric(5,2) NOT NULL,
  competition         numeric(5,2) NOT NULL,
  opportunity         numeric(5,2) NOT NULL,
  design_success_score numeric(5,2) NOT NULL,
  band                text NOT NULL,
  contributions       jsonb NOT NULL,          -- per-dimension contribution vector
  reasoning           jsonb NOT NULL,
  feature_snapshot    jsonb NOT NULL,
  scoring_config_id   uuid NOT NULL REFERENCES scoring_configs(id),
  created_at          timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX ON concept_scores (concept_id, stage, created_at DESC);

CREATE TYPE risk_level AS ENUM ('none','low','medium','high','blocked');

CREATE TABLE legal_screenings (
  id                  uuid PRIMARY KEY,
  workspace_id        uuid NOT NULL,
  subject_type        text NOT NULL,     -- concept|artwork|listing_text
  subject_id          uuid NOT NULL,
  risk_level          risk_level NOT NULL,
  extracted_entities  jsonb NOT NULL,
  copyright_assessment jsonb,
  etsy_policy_flags   text[] NOT NULL DEFAULT '{}',
  rationale           text NOT NULL,
  safer_alternatives  jsonb,
  overridden          boolean NOT NULL DEFAULT false,
  overridden_by       uuid REFERENCES users(id),
  override_justification text,
  overridden_at       timestamptz,
  screener_version    text NOT NULL,
  created_at          timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX ON legal_screenings (subject_type, subject_id, created_at DESC);

CREATE TABLE legal_matches (
  id                  uuid PRIMARY KEY,
  workspace_id        uuid NOT NULL,
  screening_id        uuid NOT NULL REFERENCES legal_screenings(id) ON DELETE CASCADE,
  matched_term        text NOT NULL,
  match_type          text NOT NULL,     -- exact|fuzzy|phonetic|blocklist
  registry            text,              -- uspto|euipo|ukipo|internal
  registration_number text,
  owner_name          text,
  nice_classes        int[],
  jurisdiction        text,
  mark_status         text,
  registry_url        text,
  similarity          numeric(5,4),
  raw                 jsonb,
  created_at          timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE artwork_briefs (
  id                  uuid PRIMARY KEY,
  workspace_id        uuid NOT NULL,
  concept_id          uuid NOT NULL REFERENCES concepts(id) ON DELETE CASCADE,
  version             int NOT NULL DEFAULT 1,
  subject             text NOT NULL,
  composition         text NOT NULL,
  palette_hexes       text[] NOT NULL,
  typography_direction text,
  texture_finish      text,
  background_requirement text NOT NULL DEFAULT 'transparent',
  aspect_ratio        text NOT NULL,
  target_print_px     jsonb NOT NULL,      -- {width, height, dpi}
  negative_constraints text[] NOT NULL DEFAULT '{}',
  pod_notes           text,
  edited_by_user      boolean NOT NULL DEFAULT false,
  prompt_version      text NOT NULL,
  created_at          timestamptz NOT NULL DEFAULT now(),
  UNIQUE (concept_id, version)
);

CREATE TYPE artwork_status AS ENUM
  ('queued','generating','generated','processing','qa_failed','ready','accepted','rejected','failed');

CREATE TABLE artworks (
  id                  uuid PRIMARY KEY,
  workspace_id        uuid NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
  concept_id          uuid NOT NULL REFERENCES concepts(id) ON DELETE CASCADE,
  brief_id            uuid NOT NULL REFERENCES artwork_briefs(id),
  variant_index       int NOT NULL,
  status              artwork_status NOT NULL DEFAULT 'queued',
  provider            text NOT NULL DEFAULT 'ideogram',
  provider_request_id text,
  prompt_text         text NOT NULL,
  negative_prompt     text,
  seed                bigint,
  parameters          jsonb NOT NULL,
  svg_ready           boolean NOT NULL DEFAULT false,
  embedding           vector(512),
  originality_score   numeric(5,2),
  nearest_competitor_listing_id uuid,
  nearest_similarity  numeric(5,4),
  originality_acknowledged_by uuid REFERENCES users(id),
  cost_amount         bigint NOT NULL DEFAULT 0,
  cost_currency       char(3) NOT NULL DEFAULT 'GBP',
  accepted_at         timestamptz,
  rejection_reason    text,
  created_at          timestamptz NOT NULL DEFAULT now(),
  updated_at          timestamptz NOT NULL DEFAULT now(),
  deleted_at          timestamptz,
  UNIQUE (concept_id, brief_id, variant_index)
);
CREATE INDEX ON artworks (workspace_id, status);
CREATE INDEX artworks_embedding ON artworks USING hnsw (embedding vector_cosine_ops);

CREATE TABLE artwork_assets (
  id             uuid PRIMARY KEY,
  workspace_id   uuid NOT NULL,
  artwork_id     uuid NOT NULL REFERENCES artworks(id) ON DELETE CASCADE,
  kind           text NOT NULL,      -- original|bg_removed|upscaled|print_png|web_png|thumb|svg|proof
  derived_from   uuid REFERENCES artwork_assets(id),
  asset_id       uuid NOT NULL,      -- -> assets
  width_px       int,
  height_px      int,
  dpi_at_target  numeric(7,2),
  has_alpha      boolean,
  colour_count   int,
  file_bytes     bigint,
  created_at     timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE artwork_qa_results (
  id             uuid PRIMARY KEY,
  workspace_id   uuid NOT NULL,
  artwork_id     uuid NOT NULL REFERENCES artworks(id) ON DELETE CASCADE,
  asset_id       uuid,
  overall        text NOT NULL,       -- pass|warn|fail
  criteria       jsonb NOT NULL,      -- [{key, status, measured, threshold, remedy}]
  qa_version     text NOT NULL,
  created_at     timestamptz NOT NULL DEFAULT now()
);
```

### 3.7 Commerce

```sql
CREATE TABLE print_providers (
  id                 uuid PRIMARY KEY,
  printify_id        int NOT NULL UNIQUE,
  name               text NOT NULL,
  location           text,
  avg_production_days numeric(5,2),
  reliability_rating numeric(3,2),
  synced_at          timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE blueprints (
  id                 uuid PRIMARY KEY,
  printify_id        int NOT NULL UNIQUE,
  product_type_id    uuid REFERENCES product_types(id),
  title              text NOT NULL,
  brand              text,
  model              text,
  print_areas        jsonb NOT NULL,
  images             jsonb,
  active             boolean NOT NULL DEFAULT true,
  synced_at          timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE blueprint_variants (
  id                     uuid PRIMARY KEY,
  blueprint_id           uuid NOT NULL REFERENCES blueprints(id) ON DELETE CASCADE,
  print_provider_id      uuid NOT NULL REFERENCES print_providers(id),
  printify_variant_id    int NOT NULL,
  colour                 text,
  colour_hex             text,
  size                   text,
  cost_amount            bigint NOT NULL,
  cost_currency          char(3) NOT NULL,
  shipping_first_amount  bigint,
  shipping_region        text,
  in_stock               boolean NOT NULL DEFAULT true,
  synced_at              timestamptz NOT NULL DEFAULT now(),
  UNIQUE (blueprint_id, print_provider_id, printify_variant_id)
);
CREATE INDEX ON blueprint_variants (blueprint_id, print_provider_id);

CREATE TYPE draft_status AS ENUM
  ('draft','printify_pending','printify_ready','etsy_pending','etsy_draft',
   'partial','review','published','failed','archived');

CREATE TABLE product_drafts (
  id                     uuid PRIMARY KEY,
  workspace_id           uuid NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
  run_id                 uuid REFERENCES runs(id) ON DELETE SET NULL,
  concept_id             uuid NOT NULL REFERENCES concepts(id),
  artwork_id             uuid NOT NULL REFERENCES artworks(id),
  blueprint_id           uuid REFERENCES blueprints(id),
  print_provider_id      uuid REFERENCES print_providers(id),
  seo_variation_id       uuid,
  fee_model_id           uuid REFERENCES fee_models(id),
  status                 draft_status NOT NULL DEFAULT 'draft',
  retail_price_amount    bigint,
  retail_price_currency  char(3) NOT NULL DEFAULT 'GBP',
  free_shipping          boolean NOT NULL DEFAULT true,
  unit_cost_amount       bigint,
  net_profit_amount      bigint,
  margin_pct             numeric(6,4),
  placement              jsonb,          -- {x,y,scale,angle,print_area}
  printify_product_id    text,
  printify_image_id      text,
  etsy_listing_id        bigint,
  checklist              jsonb,
  idempotency_key        text,
  published_at           timestamptz,
  published_by           uuid REFERENCES users(id),
  error                  jsonb,
  created_at             timestamptz NOT NULL DEFAULT now(),
  updated_at             timestamptz NOT NULL DEFAULT now(),
  deleted_at             timestamptz
);
CREATE INDEX ON product_drafts (workspace_id, status, updated_at DESC);
CREATE UNIQUE INDEX product_drafts_idem ON product_drafts (workspace_id, idempotency_key)
  WHERE idempotency_key IS NOT NULL;

CREATE TABLE product_variants (
  id                     uuid PRIMARY KEY,
  workspace_id           uuid NOT NULL,
  product_draft_id       uuid NOT NULL REFERENCES product_drafts(id) ON DELETE CASCADE,
  blueprint_variant_id   uuid NOT NULL REFERENCES blueprint_variants(id),
  sku                    text NOT NULL,
  enabled                boolean NOT NULL DEFAULT true,
  price_amount           bigint NOT NULL,
  price_currency         char(3) NOT NULL,
  cost_amount            bigint NOT NULL,
  margin_pct             numeric(6,4),
  is_default             boolean NOT NULL DEFAULT false,
  UNIQUE (product_draft_id, blueprint_variant_id)
);

CREATE TABLE product_recommendations (
  id                  uuid PRIMARY KEY,
  workspace_id        uuid NOT NULL,
  run_id              uuid NOT NULL REFERENCES runs(id) ON DELETE CASCADE,
  blueprint_id        uuid NOT NULL REFERENCES blueprints(id),
  print_provider_id   uuid NOT NULL REFERENCES print_providers(id),
  demand_score        numeric(5,2) NOT NULL,
  competition_score   numeric(5,2) NOT NULL,
  profitability_score numeric(5,2) NOT NULL,
  overall_score       numeric(5,2) NOT NULL,
  suggested_price_amount bigint,
  suggested_price_currency char(3),
  unit_cost_amount    bigint,
  margin_pct          numeric(6,4),
  recommended_colours text[] NOT NULL DEFAULT '{}',
  recommended_sizes   text[] NOT NULL DEFAULT '{}',
  reasoning           text,
  rank                int NOT NULL,
  created_at          timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE seo_variations (
  id                  uuid PRIMARY KEY,
  workspace_id        uuid NOT NULL,
  product_draft_id    uuid REFERENCES product_drafts(id) ON DELETE CASCADE,
  concept_id          uuid REFERENCES concepts(id),
  run_id              uuid REFERENCES runs(id),
  variation_index     int NOT NULL,
  axis                text NOT NULL,    -- gift|audience|humour|benefit|occasion|longtail|broad|seasonal|personalisation|premium
  title               text NOT NULL,
  description         text NOT NULL,
  tags                text[] NOT NULL,
  keywords            jsonb NOT NULL,   -- [{term, rank, evidence, competition_index}]
  positioning         text,
  materials           text[] NOT NULL DEFAULT '{}',
  taxonomy_id         int,
  quality_score       numeric(5,2),
  quality_breakdown   jsonb,
  validation          jsonb NOT NULL,   -- {title_len, tag_count, tag_len_ok, duplicates}
  is_selected         boolean NOT NULL DEFAULT false,
  edited_by_user      boolean NOT NULL DEFAULT false,
  prompt_version      text NOT NULL,
  created_at          timestamptz NOT NULL DEFAULT now(),
  updated_at          timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT tags_max_13 CHECK (array_length(tags,1) IS NULL OR array_length(tags,1) <= 13),
  CONSTRAINT title_max_140 CHECK (char_length(title) <= 140)
);
CREATE UNIQUE INDEX one_selected_seo_per_draft
  ON seo_variations (product_draft_id) WHERE is_selected;

CREATE TABLE pricing_snapshots (
  id                  uuid PRIMARY KEY,
  workspace_id        uuid NOT NULL,
  product_draft_id    uuid NOT NULL REFERENCES product_drafts(id) ON DELETE CASCADE,
  retail_price_amount bigint NOT NULL,
  currency            char(3) NOT NULL,
  breakdown           jsonb NOT NULL,   -- itemised fee stack
  net_profit_amount   bigint NOT NULL,
  margin_pct          numeric(6,4) NOT NULL,
  fee_model_id        uuid REFERENCES fee_models(id),
  created_at          timestamptz NOT NULL DEFAULT now()
);
```

### 3.8 Publishing & performance

```sql
CREATE TABLE integrations (
  id                 uuid PRIMARY KEY,
  workspace_id       uuid NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
  provider           text NOT NULL,     -- etsy|printify|ideogram|anthropic|everbee
  status             text NOT NULL DEFAULT 'disconnected',
  external_account_id text,
  external_shop_id   text,
  scopes             text[] NOT NULL DEFAULT '{}',
  quota_used         int,
  quota_limit        int,
  quota_reset_at     timestamptz,
  last_success_at    timestamptz,
  last_error         jsonb,
  breaker_state      text NOT NULL DEFAULT 'closed',
  breaker_until      timestamptz,
  connected_at       timestamptz,
  created_at         timestamptz NOT NULL DEFAULT now(),
  updated_at         timestamptz NOT NULL DEFAULT now(),
  UNIQUE (workspace_id, provider)
);

CREATE TABLE integration_credentials (
  id                 uuid PRIMARY KEY,
  integration_id     uuid NOT NULL REFERENCES integrations(id) ON DELETE CASCADE,
  workspace_id       uuid NOT NULL,
  ciphertext         bytea NOT NULL,       -- envelope-encrypted JSON
  dek_wrapped        bytea NOT NULL,
  key_version        int NOT NULL,
  expires_at         timestamptz,
  rotated_at         timestamptz,
  created_at         timestamptz NOT NULL DEFAULT now()
);

CREATE TYPE listing_state AS ENUM ('draft','active','inactive','sold_out','expired','removed');

CREATE TABLE listings (
  id                   uuid PRIMARY KEY,
  workspace_id         uuid NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
  product_draft_id     uuid REFERENCES product_drafts(id),
  concept_id           uuid REFERENCES concepts(id),
  artwork_id           uuid REFERENCES artworks(id),
  run_id               uuid REFERENCES runs(id),
  niche_id             uuid REFERENCES niches(id),
  etsy_listing_id      bigint NOT NULL,
  etsy_shop_id         bigint,
  state                listing_state NOT NULL DEFAULT 'draft',
  title                text NOT NULL,
  price_amount         bigint,
  price_currency       char(3),
  tags                 text[] NOT NULL DEFAULT '{}',
  url                  text,
  published_at         timestamptz,
  first_sale_at        timestamptz,
  last_synced_at       timestamptz,
  sync_error           jsonb,
  predicted_score      numeric(5,2),
  created_at           timestamptz NOT NULL DEFAULT now(),
  updated_at           timestamptz NOT NULL DEFAULT now(),
  deleted_at           timestamptz,
  UNIQUE (workspace_id, etsy_listing_id)
);
CREATE INDEX ON listings (workspace_id, state, published_at DESC);

CREATE TABLE listing_images (
  id             uuid PRIMARY KEY,
  workspace_id   uuid NOT NULL,
  listing_id     uuid REFERENCES listings(id) ON DELETE CASCADE,
  product_draft_id uuid REFERENCES product_drafts(id) ON DELETE CASCADE,
  asset_id       uuid NOT NULL,
  source         text NOT NULL,     -- printify_mockup|artwork|upload
  position       int NOT NULL,
  is_primary     boolean NOT NULL DEFAULT false,
  etsy_image_id  bigint,
  uploaded_at    timestamptz,
  created_at     timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE publish_jobs (
  id                 uuid PRIMARY KEY,
  workspace_id       uuid NOT NULL,
  product_draft_id   uuid NOT NULL REFERENCES product_drafts(id) ON DELETE CASCADE,
  operation          text NOT NULL,   -- printify_upload|printify_create|mockups|etsy_draft|etsy_images|etsy_inventory|etsy_publish
  status             text NOT NULL DEFAULT 'pending',
  attempt            int NOT NULL DEFAULT 0,
  idempotency_key    text NOT NULL,
  request_payload    jsonb,
  response_payload   jsonb,
  error              jsonb,
  started_at         timestamptz,
  finished_at        timestamptz,
  created_at         timestamptz NOT NULL DEFAULT now(),
  UNIQUE (workspace_id, idempotency_key)
);

CREATE TABLE performance_snapshots (
  id                 uuid NOT NULL,
  workspace_id       uuid NOT NULL,
  listing_id         uuid NOT NULL,
  captured_at        timestamptz NOT NULL DEFAULT now(),
  views              int,
  favourites         int,
  orders             int,
  revenue_amount     bigint,
  revenue_currency   char(3),
  state              listing_state,
  views_delta        int,
  orders_delta       int,
  conversion_rate    numeric(8,6),
  source             text NOT NULL DEFAULT 'etsy_api',
  PRIMARY KEY (id, captured_at)
) PARTITION BY RANGE (captured_at);
CREATE INDEX ON performance_snapshots (workspace_id, listing_id, captured_at DESC);
```

### 3.9 Learning

```sql
CREATE TABLE outcome_features (
  id                    uuid PRIMARY KEY,
  workspace_id          uuid NOT NULL,
  listing_id            uuid NOT NULL REFERENCES listings(id) ON DELETE CASCADE,
  concept_id            uuid REFERENCES concepts(id),
  run_id                uuid REFERENCES runs(id),
  features              jsonb NOT NULL,     -- flat feature vector, versioned by feature_set_version
  feature_set_version   text NOT NULL,
  outcome_window_days   int NOT NULL,
  views                 int,
  orders                int,
  revenue_amount        bigint,
  revenue_currency      char(3),
  age_normalised_percentile numeric(5,2),
  days_to_first_sale    int,
  is_success            boolean,            -- top quartile in workspace
  computed_at           timestamptz NOT NULL DEFAULT now(),
  UNIQUE (listing_id, outcome_window_days, feature_set_version)
);

CREATE TABLE prediction_records (
  id                  uuid PRIMARY KEY,
  workspace_id        uuid NOT NULL,
  concept_id          uuid NOT NULL,
  listing_id          uuid REFERENCES listings(id),
  predicted_score     numeric(5,2) NOT NULL,
  predicted_band      text NOT NULL,
  actual_percentile   numeric(5,2),
  error               numeric(6,2),
  scoring_config_id   uuid NOT NULL REFERENCES scoring_configs(id),
  evaluated_at        timestamptz,
  created_at          timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE model_proposals (
  id                  uuid PRIMARY KEY,
  workspace_id        uuid NOT NULL,
  proposed_config_id  uuid NOT NULL REFERENCES scoring_configs(id),
  base_config_id      uuid NOT NULL REFERENCES scoring_configs(id),
  method              text NOT NULL,      -- ridge|lasso|gbdt
  n_outcomes          int NOT NULL,
  weight_diff         jsonb NOT NULL,
  status              text NOT NULL DEFAULT 'proposed',  -- proposed|activated|rejected
  decided_by          uuid REFERENCES users(id),
  decision_note       text,
  decided_at          timestamptz,
  created_at          timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE model_backtests (
  id                  uuid PRIMARY KEY,
  workspace_id        uuid NOT NULL,
  proposal_id         uuid NOT NULL REFERENCES model_proposals(id) ON DELETE CASCADE,
  split               text NOT NULL,      -- train|test
  n                   int NOT NULL,
  r2                  numeric(6,4),
  brier               numeric(6,4),
  spearman            numeric(6,4),
  top_quartile_precision numeric(6,4),
  details             jsonb,
  created_at          timestamptz NOT NULL DEFAULT now()
);
```

### 3.10 Platform

```sql
CREATE TABLE assets (
  id             uuid PRIMARY KEY,
  workspace_id   uuid NOT NULL,
  bucket         text NOT NULL,
  object_key     text NOT NULL,
  content_type   text NOT NULL,
  bytes          bigint NOT NULL,
  sha256         text NOT NULL,
  width_px       int,
  height_px      int,
  purpose        text NOT NULL,   -- competitor_thumb|artwork|mockup|export|import
  created_at     timestamptz NOT NULL DEFAULT now(),
  deleted_at     timestamptz,
  UNIQUE (workspace_id, sha256, purpose)
);
CREATE INDEX ON assets (workspace_id, purpose, created_at DESC);

CREATE TABLE provider_calls (        -- partitioned monthly
  id             uuid NOT NULL,
  workspace_id   uuid NOT NULL,
  run_id         uuid,
  run_step_id    uuid,
  provider       text NOT NULL,
  operation      text NOT NULL,
  http_status    int,
  ok             boolean NOT NULL,
  latency_ms     int,
  retry_count    int NOT NULL DEFAULT 0,
  cost_amount    bigint NOT NULL DEFAULT 0,
  cost_currency  char(3) NOT NULL DEFAULT 'GBP',
  request_hash   text,
  request_body   jsonb,             -- purged after 30 days
  response_body  jsonb,             -- purged after 30 days
  error          jsonb,
  correlation_id text,
  created_at     timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);
CREATE INDEX ON provider_calls (workspace_id, provider, created_at DESC);
CREATE INDEX ON provider_calls (run_id);

CREATE TABLE ai_calls (              -- partitioned monthly
  id                 uuid NOT NULL,
  workspace_id       uuid NOT NULL,
  run_id             uuid,
  run_step_id        uuid,
  purpose            text NOT NULL,    -- subniche_discovery|style_classify|concepts|seo|legal|...
  model_tier         text NOT NULL,    -- reasoning|analysis|extraction|vision
  model_id           text NOT NULL,
  prompt_template    text NOT NULL,
  prompt_version     text NOT NULL,
  input_hash         text NOT NULL,
  cache_hit          boolean NOT NULL DEFAULT false,
  tokens_in          int NOT NULL DEFAULT 0,
  tokens_out         int NOT NULL DEFAULT 0,
  cached_tokens_in   int NOT NULL DEFAULT 0,
  cost_amount        bigint NOT NULL DEFAULT 0,
  cost_currency      char(3) NOT NULL DEFAULT 'GBP',
  latency_ms         int,
  schema_valid       boolean,
  repair_attempts    int NOT NULL DEFAULT 0,
  raw_response_ref   text,             -- object storage key; purged after 90 days
  error              jsonb,
  created_at         timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);
CREATE INDEX ON ai_calls (workspace_id, purpose, created_at DESC);
CREATE INDEX ON ai_calls (input_hash);

CREATE TABLE notifications (
  id             uuid PRIMARY KEY,
  workspace_id   uuid NOT NULL,
  user_id        uuid REFERENCES users(id),
  type           text NOT NULL,
  severity       text NOT NULL DEFAULT 'normal',
  title          text NOT NULL,
  body           text,
  link           text,
  entity_type    text,
  entity_id      uuid,
  read_at        timestamptz,
  created_at     timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX ON notifications (workspace_id, user_id, read_at NULLS FIRST, created_at DESC);

CREATE TABLE feature_flags (
  key            text PRIMARY KEY,
  enabled        boolean NOT NULL DEFAULT false,
  workspace_ids  uuid[] NOT NULL DEFAULT '{}',
  description    text,
  updated_at     timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE jobs_dlq (
  id             uuid PRIMARY KEY,
  workspace_id   uuid,
  queue          text NOT NULL,
  job_name       text NOT NULL,
  payload        jsonb NOT NULL,
  attempts       int NOT NULL,
  last_error     jsonb,
  failed_at      timestamptz NOT NULL DEFAULT now(),
  resolved_at    timestamptz,
  resolution     text
);
```

---

## 4. Key indexes (beyond those inline)

```sql
-- Hot paths
CREATE INDEX ON listing_snapshots (workspace_id, run_id, est_monthly_sales DESC NULLS LAST);
CREATE INDEX ON success_factors (workspace_id, attribute, value) WHERE NOT insufficient_evidence;
CREATE INDEX ON concepts (workspace_id, created_at DESC) WHERE deleted_at IS NULL;
CREATE INDEX ON product_drafts (workspace_id) WHERE status IN ('review','partial');
CREATE INDEX ON performance_snapshots (workspace_id, captured_at DESC);

-- Cost analytics
CREATE INDEX ON ai_calls (workspace_id, created_at DESC) INCLUDE (cost_amount);

-- Full text across owned entities (materialised search view refreshed on write)
CREATE MATERIALIZED VIEW search_index AS
  SELECT 'niche' AS kind, id, workspace_id, name AS label,
         to_tsvector('english', name || ' ' || coalesce(description,'')) AS tsv
    FROM niches WHERE deleted_at IS NULL
  UNION ALL
  SELECT 'concept', id, workspace_id, name,
         to_tsvector('english', name || ' ' || description) FROM concepts WHERE deleted_at IS NULL
  UNION ALL
  SELECT 'listing', id, workspace_id, title,
         to_tsvector('english', title) FROM listings WHERE deleted_at IS NULL;
CREATE INDEX ON search_index USING gin (tsv);
CREATE INDEX ON search_index (workspace_id, kind);
```

---

## 5. Row-Level Security (written now, enforced at SaaS phase)

```sql
ALTER TABLE runs ENABLE ROW LEVEL SECURITY;
CREATE POLICY runs_tenant_isolation ON runs
  USING (workspace_id = current_setting('app.workspace_id')::uuid);
-- …repeated for every workspace-scoped table via a migration generator.
```

The application sets `app.workspace_id` and `app.user_id` per connection checkout. In Phase 1 the policies exist but the app connects as a role with `BYPASSRLS`; flipping to an RLS-enforced role is a one-line deployment change, and the policies are tested in CI from day one.

---

## 6. Retention & lifecycle jobs

| Data | Retention | Job |
|---|---|---|
| `listing_snapshots` raw rows | 180 days (aggregates persist) | monthly partition drop |
| `provider_calls.request_body/response_body` | 30 days | nightly `UPDATE … SET body = NULL` |
| `ai_calls.raw_response_ref` objects | 90 days | nightly object lifecycle rule |
| `run_events` | 90 days | partition drop |
| `performance_snapshots` | indefinite (downsampled to weekly after 1 year) | monthly rollup job |
| Soft-deleted rows | purge at 30 days | nightly |
| `audit_log` | 7 years | none (append-only) |
| Competitor thumbnails | 180 days, then only for listings referenced by an active analysis | nightly GC by reference count |

---

## 7. Volume projections

| Table | Phase 1 (yr 1) | SaaS Stage 3 (1,000 workspaces) |
|---|---|---|
| `listing_snapshots` | 1.2 M rows / 4 GB | 1.2 B rows / 4 TB (partitioned, cold tiers) |
| `style_profiles` | 400 k / 3 GB (vectors) | 400 M / 3 TB |
| `ai_calls` | 300 k | 300 M |
| `provider_calls` | 500 k | 500 M |
| `concepts` | 20 k | 20 M |
| `artworks` | 8 k + 60 GB objects | 8 M + 60 TB objects |
| `performance_snapshots` | 150 k | 150 M |

Partitioning, cold-storage tiering and the read-replica strategy that these numbers demand are specified in [doc 16](16-scalability-architecture.md).

---

## 8. Migration policy

- **Expand/contract only.** Every migration must leave the previously deployed application version working: add nullable column → backfill → start writing → start reading → drop old column in a later release.
- **No blocking DDL in business hours.** `CREATE INDEX CONCURRENTLY`, `ALTER TABLE … ADD COLUMN` with defaults only on PG16 fast-default paths, `NOT NULL` added via `CHECK … NOT VALID` then `VALIDATE`.
- **Every migration ships with:** an up script, a documented rollback (or an explicit "forward-only, here's the compensating action"), and a test against a production-shaped seed.
- **Prisma is the schema source of truth**, but any migration that Prisma cannot express safely (partitioning, RLS, concurrent indexes, generated columns) is written as raw SQL in the migration file.
