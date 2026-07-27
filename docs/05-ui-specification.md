# 05 — UI Specification

**Version:** 1.0 · Framework: Next.js 15 App Router · React 19 · Tailwind CSS · shadcn/ui (Radix primitives) · TanStack Query + TanStack Table · Recharts · Framer Motion (restrained)

---

## 1. Design system

### 1.1 Principles

1. **Evidence adjacency.** Any number is within one click of the data that produced it. Scores are never bare.
2. **Density with hierarchy.** This is a professional analysis tool. Show a lot, but rank it — one primary metric per card, supporting detail secondary.
3. **State honesty.** Estimated data looks different from measured data. Degraded results are visibly degraded. Loading states are determinate.
4. **Spend visibility.** Any action that costs money says what it costs before it happens.
5. **Progressive disclosure.** Summary → breakdown → raw evidence, in three consistent layers everywhere.

### 1.2 Tokens

```
Colour (semantic, both themes)
  bg / bg-subtle / surface / surface-raised / border / border-strong
  text / text-muted / text-subtle
  brand           accent for primary actions and active nav
  positive        success factors, margin healthy, publish success
  caution         warnings, medium risk, degraded data
  danger          failures, blocked, high risk
  info            estimates, informational badges
  Score bands: weak · moderate · promising · strong · exceptional (5-step sequential ramp,
               distinguishable in greyscale; never colour-only — always paired with a label)

Typography
  Display 32/38 semibold · H1 24/30 · H2 20/26 · H3 16/22 semibold
  Body 14/20 · Small 13/18 · Micro 12/16 · Mono 13/18 (IDs, hex, SKUs)

Spacing        4px base scale (4 8 12 16 24 32 48 64)
Radius         sm 6 · md 10 · lg 14 · full
Elevation      0 flat · 1 card · 2 popover · 3 modal
Motion         120ms micro · 200ms panel · 320ms page; respects prefers-reduced-motion
```

### 1.3 Core component inventory

| Component | Purpose | Key states |
|---|---|---|
| `ScoreDial` | 0–100 score with band label | value, band, size, trend delta, loading |
| `ScoreBreakdown` | Sub-score bars with contribution weights | expanded/collapsed |
| `EvidenceDrawer` | Right-hand panel showing source rows for any claim | open/closed, loading, empty |
| `FactorCard` | Statement + support % + baseline + lift + n + confidence chip | confidence level, insufficient-evidence variant |
| `ConfidenceChip` | low / medium / high with tooltip explaining basis | — |
| `EstimateBadge` | Marks estimated vs measured values | source, confidence |
| `RunProgressTracker` | Vertical step list with status, duration, cost | pending/running/succeeded/failed/skipped/blocked |
| `CostMeter` | Spend vs budget with threshold colouring | under/warning/at-cap |
| `DataTable` | Virtualised, sortable, filterable, column-configurable, exportable | loading skeleton, empty, error |
| `ConceptCard` | Concept summary with scores and origin badge | unselected/selected/regenerating/legal-flagged |
| `ArtworkTile` | Variant with transparency proof toggle and QA chip | generating/ready/failed/accepted |
| `QAPanel` | Print-readiness criteria list | pass/warn/fail per criterion |
| `RiskChip` | none/low/medium/high/blocked | with matched-term count |
| `ChecklistPanel` | Hard/soft pre-publish checks | pass/fail/warn |
| `PaletteSwatch` | Colour set with proportions and family name | — |
| `MoneyValue` | Minor-unit-safe currency rendering | currency, sign, emphasis |
| `EmptyState` | Illustration + explanation + primary action | per-context copy |
| `ErrorState` | What happened / why / next action | retriable or not |
| `ProviderHealthBadge` | Connected / degraded / breaker-open / disconnected | — |

### 1.4 Universal layout

```
┌──────────────────────────────────────────────────────────────────────┐
│ TOP BAR: logo · global search (⌘K) · active-run pill · cost pill ·   │
│          notifications · integration health · theme · account        │
├───────────┬──────────────────────────────────────────────────────────┤
│ SIDEBAR   │  PAGE HEADER: breadcrumb · title · context · actions      │
│           ├──────────────────────────────────────────────────────────┤
│ Dashboard │                                                          │
│ Research  │  CONTENT                                                 │
│ Reports   │                                                          │
│ Concepts  │                                                          │
│ Artwork   │                                                          │
│ Products  │                                                          │
│ Listings  │                                                          │
│ Analytics │                                                          │
│ ────────  │                                                          │
│ Settings  │                                                          │
└───────────┴──────────────────────────────────────────────────────────┘
                                          ◀ EvidenceDrawer (overlay)
```

Sidebar collapses to icons below 1440 px. Minimum supported viewport 1280×800; below that, data tables become horizontally scrollable containers (the page body itself never scrolls horizontally).

---

## 2. Route map

```
/                                   Dashboard
/research                           Run history
/research/new                       New run wizard
/research/[runId]                   Live run monitor
/research/[runId]/opportunity       Opportunity report
/research/[runId]/competitors       Competitor analysis
/research/[runId]/competitors/[listingId]   Listing detail (modal route)
/research/[runId]/success           Success analysis
/research/[runId]/failure           Failure analysis
/research/[runId]/gaps              Market gaps
/research/[runId]/concepts          Concept board
/research/[runId]/compare/[otherRunId]      Run diff
/concepts                           All concepts
/concepts/[conceptId]               Concept detail (predictor + legal)
/artwork                            Artwork library
/artwork/[artworkId]                Artwork studio
/products                           Product drafts
/products/[draftId]                 Product builder (tabs: product · pricing · seo · printify · etsy)
/products/[draftId]/review          Final review & publish
/listings                           Published listings
/listings/[listingId]               Listing performance detail
/analytics                          Analytics home
/analytics/performance              Portfolio performance
/analytics/accuracy                 Prediction calibration
/analytics/costs                    Spend
/analytics/learning                 Scoring configs & recalibration
/niches                             Niche library (incl. rejected)
/settings/*                         account · workspace · integrations · economics · scoring · data · notifications
```

---

## 3. Page specifications

### 3.1 `/` — Dashboard

**Purpose:** answer "what should I do next?" in five seconds.

**Layout**
- **Row 1 — KPI strip (4 tiles):** Published listings (30d) · Revenue (30d, with sparkline) · Products in pipeline · Month-to-date spend vs budget.
- **Row 2 — Active work (2/3 width):** active runs with `RunProgressTracker` mini; drafts awaiting review; concepts awaiting selection; artwork awaiting acceptance. Each row is a direct link to the exact next action.
- **Row 2 — Right rail (1/3):** integration health list; recent notifications; "Start new research" primary CTA.
- **Row 3 — Performance:** revenue-over-time chart with published-listing markers; top 5 and bottom 5 listings by age-normalised performance.
- **Row 4 — Intelligence digest:** highest-scoring unexplored niches; highest-scoring unused market gaps; most recent scoring-config proposal.

**Empty state:** single centred card — "No research yet" with a three-line explanation of what a run produces and a primary CTA.

---

### 3.2 `/research/new` — New Run Wizard

Four steps, progress rail on the left, all data preserved on back-navigation.

**Step 1 — Market**
- Niche input (combobox with fuzzy match against existing niches; shows "You researched this 12 days ago — reuse or start fresh?").
- Product type (card selector with icons: T-Shirt, Sweatshirt, Hoodie, Mug, Poster/Art Print, Tote Bag).
- Optional seed keywords (tag input) and excluded terms.

**Step 2 — Style**
- Seven cards: Vintage · Typography · Hand Drawn · Illustration · Humour · Modern · **Auto Select Best Style** (visually distinct, marked "Recommended").
- Each card shows a short description and, where history exists, "Won in 4 of your last 7 runs in similar niches".

**Step 3 — Depth & budget**
- Depth selector with an explicit comparison table:

| | Quick | Standard | Deep |
|---|---|---|---|
| Shops analysed | 5 | 10 | 20 |
| Visual style analysis | ✗ | ✓ | ✓ |
| Failure analysis | ✗ | ✓ | ✓ |
| Market gaps | ✗ | ✓ | ✓ extended |
| Sub-niches | 5 | 8–12 | 12–15 |
| Est. time | ~3 min | ~9 min | ~22 min |
| Est. cost | ~£0.30 | ~£0.95 | ~£2.20 |

- Budget cap input with default from workspace settings.

**Step 4 — Confirm**
- Full summary, data sources that will be used with their current health, estimated cost and duration, and a "Start research" button showing the cost inline.

**Validation:** niche 2–80 chars; product type required; budget ≥ estimated cost (or explicit acknowledgement of a likely pause).

---

### 3.3 `/research/[runId]` — Live Run Monitor

**Left column (320 px) — `RunProgressTracker`:** every step with icon, name, status, duration, cost, and a chevron to expand into a live activity log ("Analysing shop 6 of 10 — VintageGardenCo, 34 listings"). Failed steps show the error and a Retry button.

**Main column — result cards appear as they complete**, newest first, each collapsible and linking to its full report page. Provisional results (e.g. the early opportunity score) are badged `provisional` and visibly replaced when finalised.

**Header:** run title, elapsed time, `CostMeter`, Cancel, Pause.

**Transport:** Server-Sent Events on `run.progress`; automatic reconnection with resume-from-last-event-id; falls back to 3-second polling if SSE is unavailable.

**Terminal states:** on `awaiting_selection`, a prominent banner — "20 concepts ready → Review concepts". On `failed`, an error summary with the failing step, the reason, and the recovery options.

---

### 3.4 `/research/[runId]/opportunity` — Opportunity Report

**Hero:** large `ScoreDial` (0–100) with verdict band, niche/product/style context, generation timestamp, config version, and Export.

**Sub-score panel:** radar chart (5 axes) beside five `ScoreBreakdown` rows. Each row expands to show the input features, their values, the normalisation applied, and a plain-English explanation. Each carries a `ConfidenceChip` and, where relevant, an `EstimateBadge`.

**Executive summary card:** ≤ 250 words — verdict, biggest opportunity, biggest risk.

**Sub-niche ranking table:** rank · name · opportunity score · demand · competition · example search terms · listing count observed · "Research this" action (starts a new run pre-filled with the sub-niche).

**Seasonality:** 12-month line chart with an index baseline at 100, peak/trough months annotated, and strength coefficient stated. A "best time to publish" callout derived from peak minus a configurable lead time.

**Comparison strip:** this niche's percentile against all previously researched niches in the workspace.

**Degradation banner** when applicable, naming the missing source and the one action that would fix it.

---

### 3.5 `/research/[runId]/competitors` — Competitor Analysis

**Tab A — Shops.** DataTable: shop name (link to Etsy) · age · total sales (est.) · monthly revenue (est.) · reviews · review velocity · active listings · listings analysed · selection score · selection rationale (icon → popover). Sortable, filterable, exportable.

**Tab B — Listings.** Virtualised DataTable, default 14 columns from a configurable set of 24: thumbnail · title · shop · price · shipping · free shipping · images · est. monthly sales · est. revenue · reviews · velocity · age · bestseller · palette family · typography · layout · mockup style · humour type · personalisation · tag count · title length · performance decile · confidence · source. Row click opens the listing detail modal.

**Tab C — Distributions.** Price histogram with success-cohort overlay · image-count histogram by cohort · review-velocity vs revenue scatter · shop-age vs revenue scatter · listing-age vs sales curve. All charts support brushing, which cross-filters Tab B.

**Listing detail modal:** large thumbnail with extracted `PaletteSwatch`, all captured metadata, style profile with per-attribute confidence, snapshot timeline (if captured more than once), and the raw source record behind a disclosure.

---

### 3.6 `/research/[runId]/success` — Success Analysis

**Header:** cohort definition, stated explicitly — "Top decile by age-normalised estimated monthly sales · n = 42 of 418 listings".

**Synthesis card ("The median winning listing"):** a spec sheet — modal palette family with swatches, modal typography class, modal layout archetype, median price with IQR, median image count, modal mockup style, modal garment colour. Rendered as a single scannable card, because this is the artefact the operator will actually act on.

**Weighted factors list:** ranked `FactorCard`s —

> **84%** of top-decile listings use **muted green palettes**
> Baseline 40% · **Lift 2.1×** · n = 42 · Confidence **High** · Weight 0.87
> [View the 35 listings ▸]

Filterable by attribute group (colour, typography, layout, pricing, presentation, SEO, format). Factors with `insufficient_evidence` are in a separate collapsed section, clearly labelled.

**Visual galleries:** palette gallery ranked by weight; typography distribution bars; layout archetype grid with example counts.

**Correlation matrix:** heatmap of numeric attributes × performance (Spearman), with multicollinearity flags.

**Interaction effects:** top 5 attribute pairs where combined lift exceeds the product of individual lifts.

**Auto-style decision panel** (when style = auto): the winning style with its evidence and the runners-up.

---

### 3.7 `/research/[runId]/failure` — Failure Analysis

Mirrors the success page with inverted framing plus:

**Do / Avoid sheet:** two-column side-by-side, printable, the single most-used export.

**Causality labels:** each anti-factor tagged `causal-plausible` or `correlational-only`, with a tooltip explaining the difference. Ambiguous attributes (present in both reports) appear in a dedicated section with net lift.

**"Crowded loser" panel:** attribute combinations that are both very common and heavily over-represented in failures — i.e. the trap everyone falls into.

---

### 3.8 `/research/[runId]/gaps` — Market Gaps

**Bubble map:** x = supply (competitor listing count), y = demand index, bubble size = monetisability, colour = Gap Opportunity Score. Quadrant labels: *Crowded* · *Contested* · **Sweet Spot** · *Dead*. The demand floor is drawn as a visible line with an annotation explaining why anything below it is excluded.

**Ranked gap list:** each with score dial, sub-niche, angle, style, demand evidence, supply count, explanation, suggested design angles (2–4), and a "Generate concepts for this gap" action.

**Coverage matrix:** heatmap of sub-niche × style with observed listing counts; empty cells above the demand floor are highlighted.

**Feasibility warnings:** gaps flagged as trademark-heavy, seasonally dead, or unprintable carry an explicit caution with the reason.

---

### 3.9 `/research/[runId]/concepts` — Concept Board

**Header:** counts (10 success-derived / 10 gap-derived), sort control (any predictor sub-score), filters (origin, sub-niche, style, risk), Regenerate All (with cost), Add Manual Concept.

**Grid:** `ConceptCard` — name · origin badge · `ScoreDial` (Opportunity Score) · five mini sub-score bars · style tags · 2-line description · risk chip (once screened) · select checkbox · regenerate icon.

**Selection bar (sticky bottom):** "4 concepts selected · Est. artwork cost £0.24 · Continue to legal screening →".

**Concept drawer:** full description, target audience, design angle, visual direction (palette swatches, typography class, layout archetype), text content, reasoning with the specific cited factors and gaps as clickable chips that jump to the underlying evidence, score contribution waterfall, near-duplicate warnings, and history of regenerations.

---

### 3.10 `/concepts/[conceptId]` — Concept Detail & Legal

**Left:** concept content, editable where manual.
**Right top:** Opportunity Scoring Engine — five `ScoreBreakdown` rows with contribution waterfalls showing exactly which factors added and subtracted points.
**Right bottom:** Legal & Safety panel — overall `RiskChip`, extracted entities, per-registry results table (mark, owner, registration number, class, jurisdiction, status, link), copyright risk assessment, rationale, and the disclaimer.

**Actions by risk level:** `none`/`low` → "Generate artwork"; `medium` → acknowledgement checkbox then generate; `high` → typed confirmation ("I understand the risk") + free-text justification, recorded; `blocked` → generation disabled, with safer alternatives presented as accept-and-replace cards.

---

### 3.11 `/artwork/[artworkId]` — Artwork Studio

**Layout:** left rail = brief (viewer/editor with version history); centre = canvas; right rail = QA + tools.

**Canvas:** selected variant large, with toggles for transparency checkerboard, print-area overlay, garment-colour preview (white/black/natural/heather), and zoom to 100% print pixels.

**Variant strip:** generated variants as `ArtworkTile`s with seeds, generation params, QA chip and cost.

**QA panel:** criteria list with pass/warn/fail — effective DPI at target size · alpha channel present · semi-transparent pixel % · minimum stroke width at print size · colour count · out-of-gamut % for DTG · edge halo detection · file size. Each failing criterion states the specific remedy.

**Originality panel:** nearest competitor thumbnail with similarity score and a visual side-by-side; > 0.94 requires acknowledgement before acceptance.

**Tools:** Remove background · Upscale to print size · Auto-crop · Vectorise to SVG · Download (print PNG / web PNG / SVG) · Regenerate with steer.

**Accept:** promotes the variant to `accepted` and unlocks the product step. Derived asset lineage is shown as a small tree (original → bg-removed → upscaled → vectorised).

---

### 3.12 `/products/[draftId]` — Product Builder

Tabbed, with a persistent right rail showing the running profit estimate and the pre-publish checklist state.

**Tab 1 — Product.** Ranked configuration DataTable (blueprint · print provider · unit cost · fulfilment region · production time · provider rating · demand / competition / profitability sub-scores). Selecting a row expands variant selection (colours with swatches and artwork-compatibility warnings, sizes with per-size cost).

**Tab 2 — Pricing.** Cost breakdown waterfall (Printify cost → Etsy listing fee → transaction fee → payment processing → offsite ads worst case → shipping → VAT → net). Margin target slider with live price. Competitor price distribution with the chosen price marked. Free-shipping toggle that recomputes price-inclusive economics. Margin-floor violation blocks with the exact price needed.

**Tab 3 — SEO.** Ten variation cards, each labelled with its axis and quality score; expand to edit title (counter to 140), description (structured editor with section hints), 13 tags (chip input with per-tag counter, duplicate detection), keywords with evidence links, and positioning. Etsy search-result preview. Per-variation and whole-set regeneration. Select one as the active variation.

**Tab 4 — Printify.** Upload/create status, print-area placement editor (position, scale, rotation) with a live preview, mockup gallery with drag-to-order and primary-image selection, and provider error messages mapped to remediations.

**Tab 5 — Etsy.** Taxonomy selector, attributes, shipping profile, return policy, section, who-made/when-made/is-supply, personalisation settings, inventory/variant pricing table with SKUs, and draft-creation status.

---

### 3.13 `/products/[draftId]/review` — Final Review & Publish

Single scrollable page, print-friendly.

1. **Header:** product name, Opportunity Score, status, and the Publish button (disabled until hard checks pass).
2. **Checklist panel:** hard and soft checks with pass/warn/fail and jump links to fix.
3. **Artwork:** print asset with transparency proof and QA summary.
4. **Mockups:** gallery in publish order with the primary flagged.
5. **Product:** blueprint, provider, variants table.
6. **Pricing & profit:** itemised per-unit economics and per-variant margins.
7. **SEO:** the selected variation rendered as an Etsy listing preview (search result + listing page).
8. **Legal:** screening summary with any overrides and who made them.
9. **Publish:** confirmation dialog restating shop name, listing title, price and that the listing will become publicly visible.

**Post-publish:** success state with the live Etsy link, the scheduled first performance sync, and a "Create another from this run" action.

---

### 3.14 `/listings` and `/listings/[listingId]`

**Index:** DataTable of published listings — thumbnail · title · niche · published date · price · views · favourites · orders · revenue · conversion · days-to-first-sale · age-normalised percentile · estimated Opportunity Score · prediction delta. Filter by niche, product type, concept origin, date range.

**Detail:** performance time-series (views/favourites/orders/revenue with publish and edit markers) · predicted vs actual comparison · full lineage breadcrumb (run → gap/factor → concept → artwork → draft → listing), each element clickable · Etsy state and sync status · SEO used, with the option to A/B a different stored variation.

---

### 3.15 `/analytics/*`

- **Performance:** revenue and order trends; per-niche comparison; concept-origin comparison (success-derived vs gap-derived win rates); product-type comparison; cohort survival (share of listings with a sale by day N).
- **Accuracy:** predicted score vs realised percentile scatter with a fitted line and R²; calibration curve by score band; Brier score; per-dimension predictive power ranking — i.e. which of the five sub-scores actually predicts outcomes.
- **Costs:** spend by provider/month with budget lines; cost per run, per concept, per artwork, per published product; the most expensive steps ranked; projection to month end.
- **Learning:** active scoring config with version history; proposed config diff (weight-by-weight, with direction and magnitude); back-test results on a time-split holdout; Activate / Reject with a required note; re-score-history action for comparison.

---

### 3.16 `/settings/*`

| Page | Contents |
|---|---|
| Account | Email, password, TOTP management, recovery codes, active sessions |
| Workspace | Name, base currency, country, VAT treatment, timezone |
| Integrations | Per provider: status badge, scopes, quota usage bar, last successful call, last error, connect/reconnect/disconnect, masked credential display, test-connection button |
| Economics | Fee model parameters (listing fee, transaction %, processing % + fixed, offsite ads %, regulatory fee), margin floor, default price rounding rule |
| Scoring | Active config, weight table (read-only when a fitted config is active), proposals, history, activate/revert |
| Data | Retention settings, market data adapter choice and CSV import tool, export archive, soft-delete recovery bin |
| Notifications | Per-event channel matrix (in-app / email / webhook) |
| Budgets | Per-run default, monthly cap, behaviour at cap (pause vs block) |

---

## 4. Cross-cutting UI behaviours

### 4.1 Loading
Skeletons matched to final layout (never spinners for content areas). Determinate progress for pipelines. Optimistic updates for selection, ordering and toggles, with rollback on failure and a toast explaining it.

### 4.2 Errors
Every error renders as *what happened · why · what to do*, with a correlation id copyable for support. Provider errors are mapped through a catalogue; a raw provider string is never shown as the primary message.

### 4.3 Empty states
Each has a purpose statement and one primary action. No empty grid without explanation.

### 4.4 Keyboard
`⌘K` global search/command palette · `⌘Enter` primary action on forms · `Esc` close drawer/modal · `j/k` list navigation · `?` shortcut help. All interactive elements reachable by Tab with a visible focus ring.

### 4.5 Accessibility (WCAG 2.2 AA)
Semantic landmarks; charts provide an accessible data table alternative; score bands carry text labels not just colour; live regions announce run progress; motion respects `prefers-reduced-motion`; contrast ≥ 4.5:1 verified in CI via axe-core.

### 4.6 State management
Server state via TanStack Query with per-entity keys and targeted invalidation. Client state via Zustand for wizard/selection/drawer only. URL is the source of truth for filters, sort, tab and pagination so every view is shareable and restorable. Run progress via SSE with reconnection and polling fallback.

### 4.7 Performance
Route-level code splitting; virtualised tables above 100 rows; images via `next/image` with signed URLs and responsive sizes; charts lazy-loaded; report pages served with streaming SSR and Suspense boundaries per section so a slow section never blocks the page.
