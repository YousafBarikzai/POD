# Milestone 0 — Validation Execution Plan

**Status:** Active · **Owner:** Operator (Yousaf) · **Reviewer:** Project lead
**Prerequisite:** [Phase 1D §3](../PHASE-1D-implementation-plan.md#3--milestone-0--validation-before-building)

> **This is a working document.** It defines the process, the data specification, the evaluation framework and the pass/fail criteria for Milestone 0. Results are recorded in `RESULTS.md` as we go.

---

## Contents

| Part | Section |
|---|---|
| 1 | [Overview](#part-1--overview) |
| 2 | [Full Roadmap](#part-2--full-roadmap) |
| 3 | [What I Need From You Right Now](#part-3--what-i-need-from-you-right-now) |
| 4 | [Iteration Plan](#part-4--iteration-plan) |
| 5 | [Data Collection Specification](#part-5--data-collection-specification) |
| 6 | [Analysis Methodology](#part-6--analysis-methodology) |
| 7 | [Evaluation Framework](#part-7--evaluation-framework) |
| 8 | [Pass/Fail Criteria](#part-8--passfail-criteria) |
| 9 | [Final Deliverables](#part-9--final-deliverables) |
| 10 | [Immediate Next Action](#part-10--immediate-next-action) |

---

# Part 1 — Overview

## 1.1 Purpose

Milestone 0 exists to test **one assumption** before we spend five months building on it.

Everything in POD Intelligence — the research engines, the concept generation, the scoring, the learning loop — rests on the belief that statistical analysis of competitor listings surfaces patterns a skilled seller would not otherwise see.

**If that belief is wrong, the product is not what we designed.** It becomes a publishing automation tool with a decorative research layer. Useful, but far less valuable and far more replaceable.

## 1.2 The single question

> **Does statistical analysis of competitor listings discover useful, non-obvious insights that would change what I make?**

Three words carry the weight:

| Word | Meaning | Why it matters |
|---|---|---|
| **Useful** | Relates to a decision you actually make | An interesting fact about the market that changes nothing is not useful |
| **Non-obvious** | You did not already believe it | Confirmation has value but does not justify building an analysis engine |
| **Change** | Would alter a design, price or listing decision | The test is behavioural, not intellectual |

## 1.3 Why validate before building

| | Cost |
|---|---|
| Milestone 0 | **5 days** — no code, no infrastructure |
| Discovering the same thing at Milestone 4 | **~10 weeks** of work built on a false premise |
| Discovering it after launch | The whole build |

The analysis method is fully specified in Phase 1C. Nothing about it requires software to test — cohorts, prevalence, baselines and lift are arithmetic. A spreadsheet and a careful reviewer can run the entire method by hand at small scale.

**We are testing the method, not the implementation.** If the method works by hand, building it is an engineering problem. If it does not, no amount of engineering rescues it.

## 1.4 What success looks like

You finish this exercise able to say, specifically:

> *"I did not know that muted earth palettes outperform pastels by 2× in the gardening market. I would have used pastels. I am going to change that."*

And at least three statements of that shape, across three different niches, including one market you have never worked in.

## 1.5 What failure looks like

Any of these:

- You read the findings and think *"yes, obviously"* to nearly all of them
- The findings are surprising but do not survive checking against real listings
- Nothing changes about what you would make
- The method produces incoherent results in the market you don't know — meaning it depends on you already having the answer

## 1.6 Decisions that follow

| Result | Decision |
|---|---|
| **Pass** | Proceed to Milestone 1 unchanged. Record the thresholds that produced clean results as the initial production configuration. |
| **Marginal** | Proceed, but with the weakest engine demoted and expectations reset in the product's own language. |
| **Fail — obvious findings** | Reposition. Lead with market gaps and publishing automation. Success analysis becomes supporting evidence, not the headline. |
| **Fail — noisy findings** | Raise evidence thresholds, increase sample size, re-test once. If still noisy, the visual vocabularies need refining before anything is built. |
| **Fail — incoherent on the unfamiliar niche** | Serious. The product cannot claim to work in markets you don't already know, which removes most of its value. Fundamental rethink. |

---

# Part 2 — Full Roadmap

Six stages. Roughly 5 working days of your time, spread across however long suits you.

---

## Stage 1 — Setup & Niche Selection

| | |
|---|---|
| **Goal** | Lock the three niches and the data source before any data is touched |
| **Your inputs** | Your main niche · a niche you're unsure about · **five** niches you have never worked in |
| **My work** | Review your choices for bias · **select the unfamiliar niche myself** from your five · confirm product type · confirm data track |
| **Your work** | Answer four questions honestly |
| **Deliverable** | `RESULTS.md` § Niche Selection, locked |
| **Complete when** | Three niches and one product type are fixed in writing and cannot be changed later |
| **Time** | 15 minutes |

**Why I pick the third niche:** if you choose it, you may unconsciously pick one you know a little about. Giving me five and letting me choose removes that.

---

## Stage 2 — Prior Elicitation ★ critical

| | |
|---|---|
| **Goal** | Record what you believe **before** any data exists |
| **Your inputs** | Completed `priors-form.md` — one per niche |
| **My work** | Review for vagueness and push back · lock and timestamp · commit to the repo |
| **Your work** | Write down your beliefs about palettes, typography, layout, price, images, and what fails — for all three niches, **including the one you don't know** (guess; that is the point) |
| **Deliverable** | Three completed priors forms, committed before any data is collected |
| **Complete when** | Priors are committed to git with a timestamp |
| **Time** | 45–60 minutes |

**This is the most important stage and the easiest to do badly.** Vague priors ("green probably works") cannot be scored against findings. I will send them back if they are not specific.

**Once committed, priors cannot be edited.** The git history is the proof.

---

## Stage 3 — Data Collection

| | |
|---|---|
| **Goal** | 150 listings per niche with commercial data, plus thumbnails for visual classification |
| **Your inputs** | Completed spreadsheet + numbered screenshots, per niche |
| **My work** | Validate data quality · check plausibility · flag problems · **classify all visual attributes from your screenshots** |
| **Your work** | Collect per the specification in Part 5 |
| **Deliverable** | Three validated datasets with visual attributes attached |
| **Complete when** | Each niche has ≥ 100 listings passing validation, with ≥ 90% visually classified |
| **Time** | 2–4 hours per niche |

**You do not classify visual attributes.** You capture screenshots; I classify. This removes the largest chunk of manual work and removes a bias risk — you cannot unconsciously classify toward your priors.

---

## Stage 4 — Analysis & Blind Review ★ the test

| | |
|---|---|
| **Goal** | Generate findings and get your honest reaction before you see any numbers |
| **Your inputs** | Nothing at first — then your classifications |
| **My work** | Compute cohorts, prevalence, baselines, lift, significance · suppress weak findings · **produce a shuffled statement-only list with false controls planted** |
| **Your work** | Classify every statement: *Already knew · Suspected · Surprising · Disagree*. **No numbers visible. No going back to change answers.** |
| **Deliverable** | Findings list + your classifications, both committed |
| **Complete when** | All statements classified and submitted |
| **Time** | 30–45 minutes |

**Rules for you during this stage:**
1. Do not ask me for the statistics before classifying.
2. Do not look up the listings before classifying.
3. Answer fast — first instinct. Deliberation invites rationalisation.
4. Use *Already knew* only if you would have said it **before** seeing the statement.

---

## Stage 5 — Scoring & Spot-Check

| | |
|---|---|
| **Goal** | Score the result and verify surprising findings are real |
| **Your inputs** | Spot-check results |
| **My work** | Score control discrimination · compare classifications against your locked priors · compute the validation score · reveal all statistics · select findings for spot-checking |
| **Your work** | For each *Surprising* finding, open 5 listings and confirm the pattern holds |
| **Deliverable** | Scored validation report with spot-check results |
| **Complete when** | Every surprising finding has been checked against real listings |
| **Time** | 45–60 minutes |

---

## Stage 6 — Verdict & Configuration

| | |
|---|---|
| **Goal** | A defensible pass/fail decision and the settings that carry into the build |
| **Your inputs** | Agreement or challenge on the verdict |
| **My work** | Write the final report · state the verdict against Part 8 criteria · extract initial production thresholds · list the three design decisions you would change |
| **Your work** | Read, challenge anything you disagree with, sign off |
| **Deliverable** | `RESULTS.md` complete, committed, with verdict |
| **Complete when** | The checklist in Part 9 is fully ticked |
| **Time** | 30 minutes |

---

# Part 3 — What I Need From You Right Now

## 3.1 Four answers (send in your next message)

**① Your main niche** — the market your current shop mostly sells into.
> Example: *"Gardening"*

**② A niche you are unsure about** — one you have considered but never committed to, where you genuinely don't know what wins.
> Example: *"Sea swimming / cold water swimming"*

**③ Five niches you have never worked in** — I will pick one. Give five so you cannot steer the choice.
> Example: *"Beekeeping, Morris dancing, competitive dog agility, sourdough baking, sea angling"*

**④ Which data track you can run:**

| Track | Requires | You get |
|---|---|---|
| **A** | Active EverBee subscription | Sales estimates, revenue estimates, listing age — richer cohorts |
| **B** | Nothing but a browser | Review counts and listing age as the performance proxy |

If you have EverBee, say so — we will use **Track A but record Track B fields too**, so we can also test whether the free path works. That is a genuinely valuable secondary result.

## 3.2 One decision

**Product type.** Pick **one** and use it across all three niches — mixing product types would confound every comparison.

Recommended: **T-shirts.** Highest listing volume, most design variety, most competitors, and the product type where visual attributes vary most. If your shop is primarily something else, use that instead.

## 3.3 Nothing else yet

**Do not start collecting data.** Stage 2 (priors) must be completed first, and priors written after you have looked at listings are worthless.

---

# Part 4 — Iteration Plan

Eight interactions. Roughly what each involves:

| # | You submit | I do | I return |
|---|---|---|---|
| **1** | Four answers + product type *(Part 3)* | Review for bias, pick your unfamiliar niche | Locked niche selection + three blank priors forms |
| **2** | Three completed priors forms | Review for vagueness, challenge anything unfalsifiable | Either "priors locked" or specific rewrite requests |
| **3** | *(after rewrite, if needed)* | Lock and commit priors with timestamp | Confirmation + data collection brief tailored to your three niches |
| **4** | Niche 1 spreadsheet + screenshots | Validate quality, plausibility-check, classify visuals | Data quality report — accepted, or specific fixes needed |
| **5** | Niches 2 and 3 | Same | Data quality report for both |
| **6** | Nothing | **Run the full analysis.** Compute everything. Build the shuffled statement list with controls. | Findings list — **statements only, no numbers** |
| **7** | Your four-way classification of every statement | Score controls, compare to priors, compute validation score, reveal statistics | Full scored report + list of findings to spot-check |
| **8** | Spot-check results | Final scoring and verdict | **Verdict report + production configuration + what changes** |

**Realistic elapsed time:** 5–10 working days depending on how fast you collect data. The analysis itself is quick; collection is the bottleneck.

**If data quality problems appear at interactions 4–5**, expect one extra round. That is normal and better than analysing bad data.

---

# Part 5 — Data Collection Specification

## 5.1 Sample sizes and why

| Level | Listings per niche | Total | Rationale |
|---|---|---|---|
| **Minimum acceptable** | **100** | 300 | Top decile = 10 listings. At n=10 a finding needs a very large effect to clear significance. Below this, almost nothing survives suppression and the test cannot distinguish "no patterns" from "not enough data". |
| **Recommended** | **150** | 450 | Top decile = 15. Clears the n≥8 threshold with margin. Enough that moderate effects (lift ~1.6×) become detectable. **Target this.** |
| **Ideal** | **200** | 600 | Top decile = 20. Detects subtler effects and allows the failure cohort to be analysed properly at 90+ days. Collect this if the data comes easily. |

**Do not exceed 200 per niche.** Beyond that you are spending hours for diminishing returns at this stage.

## 5.2 Shops per niche

| | Count | Rationale |
|---|---|---|
| Minimum | 6 | Below this, one shop's house style dominates and produces false "market findings" |
| **Recommended** | **8–12** | Enough diversity that the cross-shop check (a finding must appear in ≥3 shops) is meaningful |
| Maximum | 15 | More shops means fewer listings each, weakening per-shop signal |

**How to choose shops:** search Etsy for `[niche] t-shirt`, sort by relevance, and take shops appearing in the top results that have **≥ 500 shop sales** and **≥ 5 relevant listings**. Prefer shops opened in the last 3 years where you can tell — but do not agonise over it; a rough mix is fine at this stage.

## 5.3 Required fields

These are the columns in `listing-data-template.csv`. Fill what you can; blanks are acceptable on the marked fields.

| # | Field | Required | Where to find it | Notes |
|---|---|---|---|---|
| 1 | `listing_ref` | ✅ | You assign | `N1-001`, `N1-002` … `N1` = niche 1 |
| 2 | `niche` | ✅ | You assign | Must match exactly across rows |
| 3 | `shop_name` | ✅ | Listing page | |
| 4 | `listing_title` | ✅ | Listing page | Full title, exactly as shown |
| 5 | `listing_url` | ✅ | Address bar | Needed for spot-checking |
| 6 | `price` | ✅ | Listing page | Number only, no symbol |
| 7 | `currency` | ✅ | Listing page | GBP / USD |
| 8 | `shipping_cost` | ✅ | Listing page | `0` if free |
| 9 | `free_shipping` | ✅ | Listing page | Y / N |
| 10 | `image_count` | ✅ | Count the thumbnails | Critical field — one of the strongest expected signals |
| 11 | `review_count` | ✅ | Listing page | **The performance proxy on Track B** |
| 12 | `shop_total_sales` | ✅ | Shop page | The number by "Sales" |
| 13 | `shop_opened_year` | ✅ | Shop page | "On Etsy since YYYY" |
| 14 | `listing_age_months` | ⚠️ best effort | See below | Important but sometimes hard |
| 15 | `is_bestseller` | ✅ | Listing badge | Y / N |
| 16 | `tag_count` | ⚠️ optional | Not visible without tools | Leave blank on Track B |
| 17 | `description_length` | ✅ | Eyeball it | `short` <100 words · `medium` 100–300 · `long` >300 |
| 18 | `has_personalisation` | ✅ | Listing page | Y / N |
| 19 | `screenshot_ref` | ✅ | You assign | e.g. `N1-shot-03` |
| 20 | `screenshot_position` | ✅ | You assign | Position in that screenshot's grid, reading left→right, top→bottom |

### Getting `listing_age_months`

In order of preference:
1. **EverBee** shows listing age directly (Track A).
2. **Etsy listing page** — scroll to "Item details", sometimes shows a listed date.
3. **Earliest review date** — open reviews, sort oldest first. Approximates the listing's active life.
4. **Leave blank.** If more than 30% are blank in a niche, tell me — I will adjust the cohort method and note the reduced confidence.

## 5.4 Track A extra fields (EverBee only)

| Field | Notes |
|---|---|
| `est_monthly_sales` | **The performance metric.** Far better than review counts. |
| `est_total_sales` | |
| `est_monthly_revenue` | |
| `favourites` | |
| `views` | Where available |

## 5.5 Screenshots — the visual data

**This is how I classify palette, typography, layout and mockup style without you doing it manually.**

| Rule | Detail |
|---|---|
| **What to capture** | The shop's listings grid, or Etsy search results — whatever shows multiple thumbnails at once |
| **How many per shot** | 8–20 thumbnails. Whatever fits legibly. |
| **Quality** | Thumbnails must be clear enough to see colours and read text. If you have to squint, zoom in and take two shots. |
| **Naming** | `N1-shot-01`, `N1-shot-02` … one sequence per niche |
| **Mapping** | Every spreadsheet row records which screenshot it appears in and its position in that grid |
| **Order** | Record rows in the same order the thumbnails appear. Left to right, top to bottom. |

**The mapping is the part that goes wrong.** If row `N1-014` says `screenshot_ref = N1-shot-02, position = 6`, I must be able to count to the 6th thumbnail in that image and find the same product. Get this right and everything downstream works.

## 5.6 What you do **not** collect

| Not needed | Why |
|---|---|
| Palette, typography, layout, mockup style | **I classify these from your screenshots.** Doing it yourself risks classifying toward your priors. |
| Tags and keywords | Not testable at this scale without tooling |
| Descriptions in full | The length band is enough |
| Anything about your own shop | This is competitor analysis |

## 5.7 Time estimate

| Track | Per niche | Total |
|---|---|---|
| **A** — EverBee export + screenshots | 1–2 hours | 3–6 hours |
| **B** — manual collection + screenshots | 3–4 hours | 9–12 hours |

Track B is genuinely laborious. If you have EverBee, use it — but record `review_count` regardless, because comparing the two performance proxies is a valuable secondary finding.

---

# Part 6 — Analysis Methodology

## 6.1 What I compute

Exactly the method specified in Phase 1C §5, run by hand at this scale.

```
   ① Performance metric
        Track A:  est_monthly_sales
        Track B:  review_count
        ↓
   ② Age normalisation
        performance ÷ min(listing_age_months, 12)
        (skipped where age is unavailable, with confidence reduced)
        ↓
   ③ Cohort assignment
        Top decile ≥ 90th percentile     ← the success cohort
        Bottom quartile ≤ 25th, age ≥ 90 days  ← the failure cohort
        ↓
   ④ Per attribute, per value:
        cohort support   = % of top decile with this value
        baseline support = % of ALL listings with this value
        LIFT             = cohort ÷ baseline
        sample size      = count in cohort
        significance     = two-proportion test
        shop spread      = number of distinct shops contributing
        ↓
   ⑤ Suppression — a finding is discarded if ANY of:
        sample size < 8
        p > 0.10
        appears in fewer than 3 distinct shops
        ↓
   ⑥ Rank by:  |ln(lift)| × confidence × log(sample size)
```

## 6.2 Attributes analysed

| Group | Attributes |
|---|---|
| **Visual** | Palette family · dominant colour · typography style · layout archetype · mockup style · design complexity · text presence |
| **Commercial** | Price band · free shipping · shipping cost |
| **Presentation** | Image count · bestseller status |
| **Listing** | Description length · personalisation offered · title length |
| **Structural** | Shop age band · listing age band |

Roughly 15 attributes × several values each. Expect 40–80 candidate findings per niche before suppression, and 8–20 surviving.

## 6.3 How findings are challenged

Four independent guards against fooling ourselves:

| Guard | What it catches |
|---|---|
| **Baseline always computed** | "84% of winners use green" is meaningless if 82% of everything uses green. Lift is the finding; the raw percentage is noise. |
| **Cross-shop requirement (≥3 shops)** | Prevents one shop's house style being reported as a market truth. This is the guard most systems omit. |
| **Multiple-comparison disclosure** | I will tell you how many attributes were tested. Testing 60 things at p<0.10 produces ~6 false positives by chance alone. You need to know that number. |
| **Spot-check requirement** | Every surprising finding gets verified against 5 real listings before it counts. Statistics can be right and still misleading. |

## 6.4 False controls

I will plant **5 fabricated findings** in the list you review — statements that look plausible but are not supported by the data.

**Purpose: to validate you, not the system.** If you mark planted falsehoods as *Already knew*, your review is not discriminating and the whole exercise is invalid. This is a check on the measuring instrument.

You will know 5 controls exist. You will not know which.

## 6.5 Avoiding confirmation bias

Five structural defences:

| Defence | Mechanism |
|---|---|
| **Pre-registered priors** | Your beliefs are committed to git before data exists. Hindsight cannot rewrite them. |
| **You never classify visuals** | I do it from screenshots, so you cannot unconsciously label toward your expectations |
| **Statement-only review** | No numbers visible during classification, so a large lift cannot impress you into "surprising" |
| **Shuffled order** | Ranking is hidden, so you cannot infer importance from position |
| **First-instinct answering** | Deliberation invites rationalisation. Fast answers are more honest. |

---

# Part 7 — Evaluation Framework

## 7.1 The four categories

### Already knew
> *I would have stated this myself before seeing it.*

Reserve this for genuine prior knowledge. The test: **is it in your priors form?** If it is, this is legitimate. If it is not, you are probably experiencing hindsight bias — use *Suspected* instead.

**Example:** *"Listings with 3 or fewer images under-perform."* — If your priors said "image count matters, more is better", this is Already knew.

### Suspected
> *I had a hunch but no confidence, and would not have acted on it.*

The honest middle. Most real learning lands here — the finding converts a vague intuition into something specific enough to act on.

**Example:** *"The optimal price band is £21.50–£24.95."* — You may have felt "around £22 feels right" without knowing the band or that it mattered.

### Surprising
> *I did not expect this and would have guessed differently.*

The category that justifies the product.

**Example:** *"Muted earth palettes outperform pastels 2.1× in gardening."* — If your priors said pastels, this is Surprising and directly actionable.

### Disagree
> *I believe this is wrong.*

Two possible causes, and we distinguish them at spot-check:
- The finding is a false control ✅ — you caught it
- The finding is real and your prior was wrong 🔍 — which is itself a valuable result

**Never suppress a Disagree to seem agreeable.** Disagreements are the most informative answers in the whole exercise.

## 7.2 Scoring framework

### Step 1 — Control discrimination *(gate)*

```
   controls correctly marked Disagree ÷ 5

   ≥ 4/5   → review is valid, proceed
   3/5     → borderline; re-review with more care
   ≤ 2/5   → REVIEW INVALID. Redo. Not a product failure.
```

**Nothing else is scored until this gate passes.**

### Step 2 — Non-obvious rate *(primary metric)*

```
   Non-obvious rate = (Surprising + Suspected) ÷ genuine findings

   "genuine findings" = all findings minus the 5 controls
```

### Step 3 — Prior contradiction rate *(the hindsight check)*

```
   Prior contradiction = findings that contradict your written priors
                         ÷ genuine findings
```

Computed from your locked priors, independent of your classifications. **If you marked something "Already knew" but your priors said the opposite, that is hindsight bias and I will flag it.** This is why Stage 2 exists.

### Step 4 — Actionability

```
   Count of findings that would change a design, price, or listing decision.
   Target: ≥ 3, spanning at least 2 niches.
```

### Step 5 — Spot-check survival

```
   Surprising findings confirmed on manual inspection ÷ surprising findings

   Target: ≥ 70%
```

### Step 6 — Unfamiliar-niche coherence *(pass/fail, not scored)*

Did the method produce coherent, checkable findings in the niche you have never worked in?

**Binary. A failure here is disproportionately serious** — it means the product only works where you already have the answer.

### Composite validation score

```
   Validation Score =
        40 × non-obvious rate
      + 25 × min(actionable findings ÷ 5, 1)
      + 20 × spot-check survival rate
      + 15 × prior contradiction rate

   Range 0–100.
   Gated by control discrimination and unfamiliar-niche coherence —
   fail either and the score is not computed at all.
```

**Why weighted this way:** non-obvious rate is the direct measure of the assumption under test. Actionability is weighted next because a finding that changes nothing has no value. Spot-check survival guards against statistical artefacts. Prior contradiction is weighted lowest because it is a cross-check on the other measures rather than a measure in itself.

---

# Part 8 — Pass/Fail Criteria

## 8.1 Thresholds

| Threshold | Score | Non-obvious | Actionable | Spot-check | Meaning |
|---|---|---|---|---|---|
| **Fail** | < 45 | < 30% | < 3 | < 70% | The analysis mostly confirms what you knew |
| **Minimum pass** | **45–59** | **≥ 30%** | **≥ 3** | **≥ 70%** | Real value, but thinner than hoped |
| **Strong pass** | 60–74 | ≥ 45% | ≥ 5 | ≥ 75% | The thesis holds clearly |
| **Excellent** | ≥ 75 | ≥ 60% | ≥ 8 | ≥ 85% | The analysis is substantially more informative than expert intuition |

**Hard gates, regardless of score:**
- Control discrimination ≥ 4/5
- Unfamiliar-niche coherence: pass
- Zero findings you judge obviously wrong after seeing the evidence

## 8.2 If below threshold

I will diagnose **which** failure mode, because the response differs:

| Mode | Signal | Response |
|---|---|---|
| **Obvious** | High *Already knew*, low *Surprising* | Reposition the product. Gaps and automation lead; success analysis supports. Roughly 3 weeks comes out of the build. |
| **Noisy** | High *Surprising* but spot-checks fail | Raise thresholds, increase sample to 250/niche, re-test **once**. If it fails again, refine the visual vocabularies before building. |
| **Unactionable** | Findings interesting, nothing would change | Rework the synthesis output. The finding list may be right and the presentation wrong. |
| **Thin** | Fewer than 10 findings survive per niche | Increase collection volume in the build; drop the Quick depth option; set expectations accordingly. |
| **Incoherent on unfamiliar niche** | Findings make no sense in the market you don't know | **Most serious.** Fundamental reconsideration of whether the method is niche-independent. |

**In every case: fix and re-test before building.** One repeated week is cheap.

## 8.3 If it passes

1. Proceed to Milestone 1 unchanged.
2. **Record the thresholds** that produced clean results — these become the initial production configuration instead of guessed defaults.
3. **Record the most surprising findings** — these are your demo content and your proof when explaining the product to anyone.
4. Note which attributes produced the strongest findings — that informs where to invest classification effort in the build.

## 8.4 If it exceeds expectations

Score ≥ 75 means the analysis is materially more informative than expert intuition. Two consequences:

- **Consider raising the research layer's prominence** in the product. If the analysis is this strong, the reports may be more valuable than the automation.
- **Do not skip Milestone 1.** A strong result is a reason to build carefully, not quickly. The temptation to rush the foundation is exactly what §11.4 of the roadmap warns about.

---

# Part 9 — Final Deliverables

## 9.1 Checklist — "Milestone 0 complete" looks like this

**Setup**
- [ ] Three niches selected and locked, with the unfamiliar one chosen by the reviewer
- [ ] One product type fixed across all three
- [ ] Data track confirmed

**Priors**
- [ ] Three priors forms completed
- [ ] Committed to git **before** any data collection, with timestamp
- [ ] Specific enough to be falsifiable

**Data**
- [ ] ≥ 100 listings per niche passing validation (target 150)
- [ ] ≥ 6 shops per niche (target 8–12)
- [ ] All screenshots mapped correctly to spreadsheet rows
- [ ] ≥ 90% of listings visually classified
- [ ] Data quality report per niche, with plausibility issues resolved

**Analysis**
- [ ] Cohorts computed for all three niches
- [ ] Full attribute analysis with baselines and lift
- [ ] Suppression applied and the number of suppressed findings recorded
- [ ] Multiple-comparison count disclosed
- [ ] Findings ranked

**Review**
- [ ] Statement-only list issued with 5 controls planted
- [ ] All statements classified by you, blind
- [ ] Control discrimination ≥ 4/5
- [ ] Classifications compared against locked priors, hindsight flagged

**Verification**
- [ ] Every *Surprising* finding spot-checked against 5 real listings
- [ ] Every *Disagree* resolved as control-caught or prior-wrong

**Verdict**
- [ ] Validation score computed
- [ ] Verdict stated against Part 8 thresholds
- [ ] Failure mode diagnosed if below threshold
- [ ] ≥ 3 specific design decisions you would change, written down
- [ ] Production thresholds extracted
- [ ] Go / no-go decision recorded and signed off

## 9.2 Files that exist at the end

```
docs/milestone-0/
├── EXECUTION-PLAN.md              this document
├── RESULTS.md                     the running record and final verdict
├── priors/
│   ├── niche-1-priors.md          locked, timestamped
│   ├── niche-2-priors.md
│   └── niche-3-priors.md
├── data/
│   ├── niche-1-listings.csv       your collected data + my classifications
│   ├── niche-2-listings.csv
│   ├── niche-3-listings.csv
│   └── data-quality-report.md
├── analysis/
│   ├── findings-full.md           every finding with full statistics
│   ├── findings-suppressed.md     what was discarded and why
│   └── blind-review-list.md       the statement-only list, with controls marked post-hoc
├── review/
│   ├── classifications.md         your four-way answers
│   └── spot-checks.md             verification results
└── VERDICT.md                     the decision and what follows
```

---

# Part 10 — Immediate Next Action

## 10.1 Send me this now

Four answers, in one message. Plain text is fine.

```
1. MAIN NICHE:
   [the market your shop mostly sells into]

2. UNSURE NICHE:
   [one you've considered but never committed to]

3. FIVE NICHES I'VE NEVER WORKED IN:
   [a] [b] [c] [d] [e]

4. DATA TRACK:
   [A = I have EverBee  |  B = browser only]

5. PRODUCT TYPE:
   [T-shirts recommended — or your shop's main product]
```

## 10.2 Format

Plain text in the chat. No files, no spreadsheets, no screenshots yet.

## 10.3 How much data to collect before returning

**None.** Collecting data before your priors are locked would invalidate the exercise.

## 10.4 What happens next

I will:
1. Review your choices for bias — particularly whether the "unsure" niche is genuinely unsure.
2. **Pick your third niche** from your five.
3. Confirm the product type works across all three.
4. Send you **three priors forms** to complete.

Then Stage 2 begins, and only after your priors are committed to git do you touch any data.

---

## Ground rules for this exercise

| Rule | Why |
|---|---|
| **Answer honestly, not helpfully** | If you shade answers toward "this product is good", you learn nothing and waste five months. I would rather this fail now. |
| **Do not look ahead** | Do not read the findings before classifying. Do not check statistics before answering. |
| **Do not edit priors** | Once committed, they stand. That is what makes the result defensible. |
| **Tell me when something is hard** | If a data field is impossible to get, say so and I will adjust the method rather than have you guess. |
| **Challenge me** | If a finding looks wrong, say so. *Disagree* is a valid and valuable answer. |

**My commitment:** I will report the result as it comes out. If this fails, I will say so plainly and tell you what to change. A validation exercise that always passes is not a validation exercise.
