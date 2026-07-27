# POD Intelligence — Part 1: Product Definition

**Document type:** Product Requirements Document (PRD)
**Phase:** 1 of the architecture series — *product understanding only*
**Version:** 1.0
**Status:** For engineering review
**Author:** Product / Founder

> **Scope note for the engineering team.** This document defines *what* the product is, *who* it serves, and *what it must do*. It deliberately contains no database design, no API design, no technology choices and no code. Those are Parts 2 onwards. Where a requirement has technical implications, they are stated as *constraints on the solution*, not as solutions.

---

## Table of contents

1. [Executive Summary](#1-executive-summary)
2. [Product Vision](#2-product-vision)
3. [Target Users](#3-target-users)
4. [User Problems](#4-user-problems)
5. [Complete User Journey](#5-complete-user-journey)
6. [Functional Requirements](#6-functional-requirements)
7. [Non-Functional Requirements](#7-non-functional-requirements)
8. [User Stories](#8-user-stories)
9. [MVP Definition](#9-mvp-definition)
10. [Future Expansion Plan](#10-future-expansion-plan)
11. [Feature Prioritisation](#11-feature-prioritisation)
12. [Product Differentiation](#12-product-differentiation)
13. [Success Metrics](#13-success-metrics)

---

# 1. Executive Summary

## 1.1 What this product is

**POD Intelligence is an AI-powered personal automation platform that turns Etsy print-on-demand from a creative gamble into an evidence-driven process.**

A user types a niche — "Gardening" — and a product type — "T-Shirts". Within minutes the system tells them whether that market is worth entering, who is currently winning in it, what those winners have visually and commercially in common, what reliably fails, and where the unserved space is. It then generates original design concepts derived from that evidence, screens them for legal risk, renders print-ready artwork, writes SEO-optimised listings, builds the product in Printify, creates an Etsy draft, and — after the user approves it — publishes.

Afterwards it watches what actually sold, and uses that to get better at predicting what will sell next.

## 1.2 The problem in one paragraph

Print-on-demand has near-zero production cost, which means the constraint is not making products — it is **knowing which products to make**. The overwhelming majority of POD sellers choose niches on intuition, design to their own taste, imitate whichever listing happens to rank on page one, and write listings by copying a competitor's title. They then discover that roughly 3% of their listings produce 90% of their revenue, and they cannot explain why. The data needed to do better already exists in public and semi-public form, but it is scattered, unstructured, and — critically — **never connected back to the design decision itself.**

## 1.3 What makes this different

Every existing tool owns one link in the chain:

- Research tools (EverBee, eRank, Alura) give you **raw numbers** and stop.
- Image generators (Midjourney, Ideogram) give you **pictures** with no market context.
- Design template tools (Placeit, Kittl) give you **templates** everyone else also uses.
- Fulfilment tools (Printify, Printful) give you **production** and nothing else.

POD Intelligence owns **the whole chain**, and the chain is where the value compounds — because every published product feeds a growing dataset linking *design attributes → market conditions → realised sales*. That dataset cannot be bought, improves with every use, and is the durable asset behind the product.

## 1.4 Who it is for

**Initially:** one user — the creator — running one Etsy shop. The product must be complete and genuinely valuable for that single user, not a demo of a future SaaS.

**Eventually:** solo POD sellers, scaling multi-shop sellers, and agencies managing POD for clients. Every product decision in this document is made for the single user first, but is checked against whether it would obstruct that future.

## 1.5 What success looks like

| Dimension | Today (manual) | Target |
|---|---|---|
| Time to decide whether a niche is worth entering | 2–4 hours of manual research | **Under 5 minutes** |
| Time from idea to published product | 2–3 hours | **Under 15 minutes of attention** |
| Share of published listings that make at least one sale in 90 days | ~20% | **45%+** |
| Revenue per published listing | baseline | **2× baseline** |
| Confidence in *why* a product was created | intuition | **A written, evidence-backed rationale** |

## 1.6 The single sentence

> **Stop guessing which designs will sell. Find out first, then make those.**

---

# 2. Product Vision

## 2.1 Vision statement

> **POD Intelligence exists to make every design decision an informed one.**
>
> In three years, a POD seller using this product should be unable to remember what it felt like to publish a design and simply hope. They should expect, as a matter of routine, to know the market's size, its competitive shape, its visual language, its failure modes and its gaps *before* they commit a single hour of creative work — and to have a system that gets measurably better at predicting outcomes the longer they use it.

## 2.2 The strategic insight

The valuable thing here is not the AI. Anyone can call an image model.

The valuable thing is the **closed loop**:

```
Market evidence  →  Design decision  →  Published product  →  Real outcome
       ↑                                                            │
       └──────────────── the system learns ─────────────────────────┘
```

Almost nobody closes that loop. Research tools stop at "here is the data". Design tools start at "here is a blank canvas". Nothing connects the two, and *nothing at all* feeds the result back.

Once the loop is closed, the product accumulates something no competitor can copy or purchase: a longitudinal record of *"in this specific sub-niche, this palette, this typography, this layout, this price, this listing structure produced this outcome."*

That record is:
- **Impossible to buy** — it only comes from having published and measured.
- **Compounding** — it improves monotonically with use.
- **A defensive moat** — a new entrant starts at zero outcomes; this product starts with everything it has already seen.

## 2.3 Design philosophy

Seven principles that govern every product decision. When two requirements conflict, these resolve them.

### P1 — Evidence over opinion
Every recommendation shows the data that produced it. A score without a "why" is a defect, not a feature. If the system says "84% of top performers use muted green palettes", it must also show the baseline (what % of *all* listings use them), the sample size, and how confident it is.

### P2 — Cheap before expensive
Text concepts cost pennies. Artwork costs real money. Therefore: always generate concepts first, always let the human choose, and only then render artwork. Never spend image-generation budget on an unvalidated idea.

### P3 — The human owns anything irreversible
Publishing to a live shop, spending money, and accepting legal risk are human decisions. Everything else is automated. This is not timidity — it is the correct allocation of judgement.

### P4 — Original by construction
The system analyses competitors **statistically** and never passes competitor imagery or text into a generative prompt. Style is described in the abstract ("muted earth palette, condensed slab serif, centred badge lockup"), never by reference to a specific existing design. Originality is a property of the architecture, not a promise in the marketing copy.

### P5 — Degrade, never die
If a data source is unavailable, the system produces a smaller, clearly-labelled result — not an error. The user always gets something useful, and always knows exactly what is missing and what would fix it.

### P6 — Honesty about uncertainty
Estimated data must look different from measured data. Low-confidence conclusions must be labelled low-confidence. A confident-looking report built on twelve data points is worse than no report, because it actively misleads.

### P7 — Speed is a feature
The user must see meaningful partial results within seconds, not stare at a spinner for nine minutes. Every stage that completes should render immediately rather than waiting for the whole process.

## 2.4 What this product is *not*

Stating this explicitly, because the adjacent-feature pull is strong and each of these is individually reasonable:

- ❌ Not a general-purpose AI art generator
- ❌ Not a marketplace-agnostic e-commerce platform (Etsy first, deliberately)
- ❌ Not an order-fulfilment or customer-service tool
- ❌ Not an advertising-management platform
- ❌ Not a design template library
- ❌ Not a "post 500 listings a day" spam engine — the product's entire premise is that *fewer, better-chosen* products outperform volume

---

# 3. Target Users

## 3.1 Primary persona — "Sam", the Solo POD Operator

**This is the only user the MVP is built for.**

| Attribute | Detail |
|---|---|
| **Role** | Runs one Etsy POD shop, alone |
| **Experience** | 1–3 years selling on Etsy |
| **Portfolio** | 150–600 active listings |
| **Revenue** | £1,000–£8,000/month |
| **Time commitment** | 25–40 hours/week |
| **Time allocation** | ~60% research and design, ~40% operations |
| **Technical comfort** | Confident with SaaS tools (EverBee, Canva, Printify). Not a developer. Will not read documentation. |
| **Tools today** | EverBee or eRank, Canva or Photoshop, Printify, a spreadsheet, sometimes Midjourney |
| **Budget for tools** | £50–£200/month, already spending most of it |

**Goals**
1. Increase revenue without proportionally increasing hours worked.
2. Stop wasting creative effort on designs that were never going to sell.
3. Find niches before they become saturated.
4. Build a shop with a defensible position rather than a pile of random products.

**Frustrations, in their own words**
> *"I have no idea which of my next ten designs will be the one that works, so I make all ten and hope."*

> *"I spend three hours researching a niche and I still don't really know if it's a good one — I just know it looks busy."*

> *"I can see that a shop is doing well. I cannot see **why**."*

> *"I write ten listings and they all sound the same because I run out of ways to say 'gardening t-shirt'."*

> *"Half my listings have never had a single sale and I don't know what they have in common."*

**What success looks like for Sam**
Publishing 30% *fewer* products but earning 2× more, because every product was chosen with evidence — and being able to explain, for any listing in their shop, why it exists.

**How they will use the product**
Daily. Typically: one research run in the morning to evaluate a niche idea, concept review over coffee, artwork generation and publishing in the afternoon.

---

## 3.2 Secondary persona — "Jordan", the Scaling Seller

*Not served by the MVP. Considered so the product does not paint itself into a corner.*

| Attribute | Detail |
|---|---|
| **Role** | Runs 3–8 shops, or one large shop, with 1–2 virtual assistants |
| **Revenue** | £15,000–£60,000/month |
| **Portfolio** | 2,000+ listings |
| **Constraint** | Their own judgement is the bottleneck — they cannot delegate intuition |

**Goals**
1. Systematise research so a VA can execute what the owner decides.
2. Maintain quality and brand consistency across a team.
3. Identify which shops and niches deserve more investment.

**Frustrations**
> *"I know what works, but I can't transfer that knowledge to my assistant. Every time I delegate design decisions, quality drops."*

> *"I'm managing four shops in four spreadsheets and I've lost track of what I've already tested."*

**What they need that the MVP does not provide**
Shareable reports, saved research that a team member can act on, an approval queue, role separation, bulk operations, and cross-shop comparison.

**Why this matters now:** the *approval gate* the MVP builds for safety reasons (P3) turns out to be exactly the primitive a team needs for delegation. Designing it well now costs nothing extra and unlocks this persona later.

---

## 3.3 Tertiary persona — "Alex", the Agency / Brand Studio

*Future SaaS customer. Highest revenue per account, longest sales cycle.*

| Attribute | Detail |
|---|---|
| **Role** | Manages POD product lines for multiple clients |
| **Clients** | 4–15 concurrent |
| **Revenue model** | Retainer plus revenue share |
| **Constraint** | Must justify creative decisions to clients who are paying for them |

**Goals**
1. Separate each client's research and assets completely.
2. Produce professional, branded reports that justify the strategy to a client.
3. Prove ROI with data rather than assertion.

**Frustrations**
> *"My clients ask 'why this design?' and my honest answer is 'experience'. That is not a satisfying answer when they're paying me a retainer."*

**What they need**
Separate workspaces per client, white-labelled report exports, seat-based access, audit trails, and eventually programmatic access.

---

## 3.4 Acquisition persona — "Riley", the Data-Curious Newcomer

*The likely entry point for a future SaaS funnel.*

| Attribute | Detail |
|---|---|
| **Role** | Has no shop yet, or a shop with under 20 listings |
| **Revenue** | £0–£300/month |
| **State of mind** | Overwhelmed by conflicting YouTube advice; paralysed on niche choice |

**Goals**
1. Choose a first niche without wasting six months on the wrong one.
2. Understand what "good" looks like before attempting it.

**Frustrations**
> *"Every guru says a different niche is the one. I have no way to check any of it."*

**Why they matter strategically**
The **Opportunity Report alone** — the go/no-go verdict on a niche — is a complete, valuable product for this persona, and it is the cheapest part of the system to run (no image generation). It is therefore the natural low-cost entry tier in a future SaaS, and the natural marketing artefact: publishing real opportunity reports publicly demonstrates the entire value proposition.

---

## 3.5 Explicit non-users

Stated so the team can push back on requests that would distort the product:

| Not for | Why |
|---|---|
| Print-on-demand *manufacturers* | The product serves sellers, not producers |
| Sellers of handmade/physical inventory | The economics and workflow are fundamentally different |
| High-volume listing spammers | The product's premise is fewer, better products; serving this user would destroy its value proposition |
| Non-Etsy marketplace sellers (initially) | Etsy's specific mechanics — 13 tags, review-driven ranking, taxonomy — are baked into the analysis |
| Enterprise brands | Their constraints are brand governance and compliance, not market discovery |

---

# 4. User Problems

Each problem is stated, quantified where possible, and mapped to the capability that addresses it.

## 4.1 Problem: Niche selection is a coin flip

**Current state.** The seller has an idea ("fishing hoodies"), searches Etsy, sees results, and forms an impression. That impression conflates *volume of listings* with *demand*, and *number of competitors* with *difficulty*. They cannot see sales, only listings. They cannot see trend, only a snapshot.

**Cost.** 2–4 hours per niche investigated, and — far more expensively — weeks of production effort spent in markets that were never viable.

**Why manual research fails.** The critical numbers (actual sales, revenue, trend direction, seasonality, achievable margin) are either unavailable, scattered across three tools, or require manual assembly the seller does not have time for. Even with EverBee open, converting twenty data points into a go/no-go decision is a judgement call made under fatigue.

**Addressed by:** Opportunity Report Engine → a single 0–100 score with five explained sub-scores and a plain verdict.

---

## 4.2 Problem: Competitors are visible but not legible

**Current state.** The seller can see that a shop is successful. They cannot see *why*. They browse listings, form vague impressions ("they use a lot of green"), and act on those impressions without knowing whether green is actually associated with success or is simply common across the whole niche.

**Cost.** Design decisions based on pattern-matching against a handful of listings the seller happened to look at — a sample of maybe 15, chosen by whatever Etsy's algorithm surfaced that day.

**Why manual research fails.** Humans cannot compute base rates by eye. Noticing that "84% of top sellers use muted greens" requires counting 400 listings. Noticing that this is meaningful requires also counting what percentage of *all* listings use muted greens — and if that number is also 80%, the observation is worthless. Nobody does this manually, so nobody knows.

**Addressed by:** Competitor Analysis Engine + Success Analysis Engine → structured data on every relevant listing, with lift calculated against a baseline.

---

## 4.3 Problem: Nobody studies failure

**Current state.** Sellers study winners exclusively. This is survivorship bias in its purest form, and it produces a systematic blind spot: the things winners do that *everyone* does (and which therefore explain nothing), and the things losers do that are actively harmful.

**Cost.** Repeating the entire market's mistakes. Pricing too low because it feels safe. Using four images because it seems like enough. Using six tags out of thirteen.

**Why manual research fails.** Failed listings are invisible. They do not rank, so they never appear in a search. Finding them requires deliberately enumerating a shop's full catalogue and identifying the non-performers — an activity with no intuitive appeal and considerable effort.

**Addressed by:** Failure Analysis Engine → an explicit "avoid these" list with the same statistical rigour as the success list.

---

## 4.4 Problem: Everyone designs for the same crowded sub-segment

**Current state.** Search "gardening t-shirt" and the results are overwhelmingly flowers, "plant lady", and watering cans. Meanwhile composting, hydroponics, greenhouse growing, seed saving and allotment culture each have real, identifiable audiences and almost no dedicated products.

**Cost.** Competing head-on in the most crowded corner of a market while adjacent, winnable space sits empty.

**Why manual research fails.** Gaps are defined by *absence*, and absence is invisible in a search interface. You cannot search for what is not there. Identifying a gap requires mapping the full space of possible sub-segments and then measuring coverage against demand for each — a systematic exercise no seller performs.

**Addressed by:** Market Gap Engine → a demand-versus-supply map with ranked, scored opportunities, and a critical guard: a gap with no demand is a desert, not an opportunity, and must never be recommended as one.

---

## 4.5 Problem: Creative work is disconnected from market evidence

**Current state.** Even a seller who has done good research then sits down and designs from taste. The research and the creation are separate activities separated by hours, and the insight evaporates in between.

**Cost.** Research effort that does not change the output. The seller ends up making what they were always going to make.

**Why manual research fails.** There is no mechanism to carry a statistical finding into a creative brief. "Muted greens perform 2.1× better here" is a sentence in a notebook, not an input to a design tool.

**Addressed by:** Concept Generation grounded directly in the weighted success factors and market gaps, with each concept citing the specific evidence that produced it.

---

## 4.6 Problem: AI design tools have no market context

**Current state.** A seller uses an image generator and gets a technically competent image reflecting the model's aesthetic priors — which are global, generic, and identical for every user who types a similar prompt.

**Cost.** Attractive designs that no specific audience wants, and a growing homogeneity across the whole category as everyone uses the same tools with the same default prompts.

**Why manual research fails.** The seller does not know how to translate market findings into a generation prompt, and the tool has no way to receive that context even if they did.

**Addressed by:** Artwork briefs constructed from the run's actual style statistics — explicit palettes derived from what wins in *this* niche, typography classes derived from measured performance, layout archetypes derived from the success cohort.

---

## 4.7 Problem: Generated artwork is rarely actually printable

**Current state.** The seller generates an image, uploads it to Printify, and discovers it has a white background, insufficient resolution, hairline strokes that will not print, or 400 colours. They then spend 40 minutes in Photoshop, or give up and regenerate.

**Cost.** 30–60 minutes per design in rework, or silent quality failures that reach customers as refunds and bad reviews.

**Addressed by:** An automated print-readiness pipeline — background removal with edge refinement, upscaling to 300 DPI at the true print size, and a QA report that names the specific failing criterion and the specific remedy.

---

## 4.8 Problem: Legal risk is discovered too late

**Current state.** The seller creates a design, publishes it, and receives a takedown notice — or worse, a shop suspension. Trademark and copyright checking is manual, tedious, and therefore skipped.

**Cost.** Catastrophic and asymmetric. A single suspension can end a business built over years. The expected cost of skipping the check is low most of the time and unbounded occasionally.

**Why manual research fails.** Checking every design against USPTO, EUIPO and UKIPO across the relevant classes is 20 minutes of work per concept that produces "nothing found" 95% of the time. Human beings do not sustain that discipline.

**Addressed by:** A mandatory Legal & Safety gate that runs **before** any artwork is generated, cannot be bypassed, and produces safer alternatives rather than simply blocking.

---

## 4.9 Problem: SEO is guesswork dressed as technique

**Current state.** The seller copies a competitor's title structure, reuses the same nine tags across every listing, and writes descriptions that repeat the title. Tags are chosen by intuition about what buyers "probably" search.

**Cost.** Listings that never surface. A listing with zero views after 14 days is an SEO failure, and it is the most common failure mode in POD.

**Why manual research fails.** Determining which keywords actually correlate with sales requires cross-referencing every competitor's tags against their sales performance — hundreds of data points, weighted, deduplicated and ranked.

**Addressed by:** SEO Engine generating ten deliberately differentiated listing variations, with keywords selected by measured, sales-weighted performance and every keyword showing its evidence.

---

## 4.10 Problem: Publishing is 45 minutes of tedium per product

**Current state.** Upload artwork to Printify → configure the blueprint → pick a provider → select variants → set prices → wait for mockups → download mockups → create the Etsy listing → upload images in order → type the title → paste the description → enter thirteen tags one at a time → set taxonomy → set shipping → set attributes → set variants and prices again.

**Cost.** 30–45 minutes per product, entirely mechanical, entirely error-prone, and a hard ceiling on how many products a person can publish per day.

**Addressed by:** End-to-end publishing automation with a single human approval gate — reducing 45 minutes of clicking to one review screen and one confirmation.

---

## 4.11 Problem: Nothing learns

**Current state.** The seller publishes 200 products over a year. Some sell. Nothing anywhere records *why*. The seller's intuition improves slightly and unreliably; the system they use improves not at all.

**Cost.** Every seller re-learns the same lessons from scratch, slowly, and forgets most of them.

**Addressed by:** The learning loop — tracking every published product's real performance, linking it to the design and listing attributes that produced it, and periodically recalibrating the system's own predictions against reality.

---

## 4.12 Summary: why a tool, not more effort

| Problem | Why more human effort will not solve it |
|---|---|
| Niche selection | The data is not assembled anywhere a human can read it in one place |
| Competitor legibility | Base rates cannot be computed by eye across 400 listings |
| Failure blindness | Failed listings are structurally invisible in a search interface |
| Market gaps | You cannot search for absence |
| Research→design gap | No mechanism exists to carry a statistic into a creative brief |
| Print readiness | Requires per-image technical validation nobody performs consistently |
| Legal risk | Requires disciplined checking with a 95% "nothing found" rate |
| SEO | Requires cross-referencing hundreds of tags against sales |
| Publishing | Irreducibly mechanical; the only fix is automation |
| Learning | Requires longitudinal record-keeping no individual maintains |

**Every one of these is a task where the bottleneck is systematic processing, not insight.** That is precisely the class of problem software should own — and it is why the answer is a tool rather than more discipline.

---

# 5. Complete User Journey

The journey has twelve stages. For each: what the user does, what the system does, what is produced, and what the user must decide.

**Overall shape:**

```
  SETUP           RESEARCH & ANALYSIS              CREATION              PUBLISH & LEARN
    │                     │                            │                       │
 Stage 1  →  Stages 2,3,4,5  →  Stages 6,7,8,9  →  Stages 10,11  →  Stage 12
 one-time      ~9 minutes          ~6 minutes         ~3 minutes        ongoing
                    ▲                    ▲                  ▲
                    │                    │                  │
              [decision:            [decision:         [decision:
             proceed or          which concepts]      publish or not]
              reject niche]
```

**Three human decision gates** (Stage 5, Stage 7, Stage 10) are mandatory and cannot be automated away. Everything between them runs unattended.

---

## Stage 1 — Account and Setup

**Frequency:** Once. Target duration: under 10 minutes.

### User actions
1. Creates an account with email and password, and enrols a second authentication factor.
2. Names their workspace and sets base currency, country and tax status.
3. Connects their Etsy shop by authorising access.
4. Connects their Printify account and selects which Printify shop corresponds to their Etsy shop.
5. Connects the AI artwork generation service.
6. Chooses how market data will be supplied — importing an export from their existing research subscription, or proceeding with public marketplace data only.
7. Sets economic parameters: marketplace fee assumptions, minimum acceptable profit margin, monthly spending budget, and per-run spending cap.

### System actions
- Creates the workspace and initialises it with a default set of scoring weights derived from expert priors.
- Verifies each connection with a live test call and reports the specific result of each.
- Retrieves and caches the product catalogue (available products, print providers, production costs) in the background.
- Assesses which market-data capabilities are available and records what analysis will consequently be possible.
- Presents a setup summary showing exactly what is connected, what is not, and what each gap means for functionality.

### Expected output
A configured workspace with a clear connection-health display, and an explicit statement of which capabilities are available versus degraded.

### User decisions
- Which market data source to use, given an honest trade-off explanation of fidelity versus effort.
- Their minimum acceptable margin — this becomes a hard constraint later.
- Their monthly spending limit.

### Design requirements
- **Setup must be skippable and resumable.** A user who cannot immediately generate a Printify token must be able to explore the research features and connect fulfilment later.
- **Failures must be diagnostic, not generic.** "Invalid token" is unacceptable; "This token is valid but has no connected shops — connect a shop in Printify first" is required.
- **The product must be usable in research-only mode**, with publishing features clearly marked unavailable and the reason stated.

---

## Stage 2 — Niche Selection

**Frequency:** Per research run. Target duration: under 90 seconds.

### User actions
1. Enters a niche as free text — "Gardening", "Fishing", "Dog Owners", "Travel".
2. Selects a product type: T-Shirt, Sweatshirt, Hoodie, Mug, Poster / Art Print, or Tote Bag.
3. Optionally selects a design style: Vintage, Typography, Hand Drawn, Illustration, Humour, Modern, or **Auto-Select Best Style**.
4. Optionally chooses research depth: Quick, Standard, or Deep.
5. Optionally supplies seed keywords to focus the research and excluded terms to avoid.
6. Reviews the pre-flight estimate and starts the run.

### System actions
- Normalises the niche name and checks it against previously researched niches, offering to reuse an existing one rather than creating a near-duplicate.
- If the niche was researched recently, surfaces that fact with the date and prior result, and offers to view the existing report, refresh it, or start fresh.
- Calculates an estimate of duration and cost based on the chosen depth.
- Displays which data sources will be used and their current health.
- Validates the run against the remaining budget.

### Expected output
A confirmed research run, queued and started, with a stated expected duration and cost.

### User decisions
- **Style: chosen or automatic.** Auto-Select is presented as the recommended default, because the whole premise of the product is that the market should decide this rather than the user's taste.
- **Depth,** which is a genuine trade-off: Quick is 5 shops in ~3 minutes for ~£0.30; Standard is 10 shops in ~9 minutes for ~£0.95; Deep is 20 shops in ~22 minutes for ~£2.20. The interface must present this as a clear comparison table, not a dropdown.

### Design requirements
- Free-text niche entry with intelligent matching against existing niches — the user must never accidentally create "gardening", "Gardening" and "garden" as three separate markets.
- **The cost of the action must be visible before the action is taken.** No user should ever be surprised by a charge.
- Depth options must state their concrete consequences (how many shops, which analyses are included, how long, how much), not abstract labels.

---

## Stage 3 — Market Research

**Frequency:** Automatic within a run. Target duration: 2–4 minutes for this stage.

### User actions
Primarily observation. The user may:
- Watch live progress as each step completes.
- Read partial results as they become available.
- Cancel the run if early signals are clearly poor.
- Raise the budget if the run pauses at its cap.

### System actions
1. **Expands the niche into sub-niches.** For "Gardening", identifies segments such as Greenhouse Growing, Organic Gardening, Vegetable Gardening, Composting, Houseplants, Allotments, Seed Saving, Hydroponics — each with a description, rationale and example search terms. Sub-niches are derived from three independent sources (domain reasoning, co-occurrence patterns in observed listing titles and tags, and marketplace taxonomy signals), and each records which sources supported it.
2. **Estimates demand** for the niche and each sub-niche.
3. **Assesses competition** — saturation, entrenchment of incumbents, the review "moat" held by top performers, and price compression.
4. **Determines trend direction** from momentum in review accumulation and the rate of new listings and new entrants.
5. **Calculates achievable profitability** using real production costs and the user's own fee model — not an estimate.
6. **Maps seasonality** across twelve months, identifying peaks, troughs and the strength of the seasonal effect.
7. Ranks all sub-niches by their own opportunity scores.

### Expected output
- A demand assessment with its supporting evidence.
- A competition assessment with its supporting evidence.
- A trend direction with the method used to determine it.
- A profitability figure derived from real costs.
- A twelve-month seasonality profile.
- A ranked list of 5–15 sub-niches.

### User decisions
None yet — this stage feeds Stage 5.

### Design requirements
- **Progress must be informative, not decorative.** "Analysing shop 6 of 10 — 34 listings found" rather than a percentage bar.
- **Partial results appear as they complete.** The user should be reading sub-niches while competitor collection is still running.
- **Every estimated figure must be visibly labelled as an estimate**, with its source and confidence.
- **Cancellation must be immediate and lossless** — completed work is retained, spending stops.

---

## Stage 4 — Competitor Analysis

**Frequency:** Automatic within a run. Target duration: 3–5 minutes.

### User actions
Observation during the run. Afterwards, deep exploration:
- Sorting and filtering the competitor listing table across 20+ attributes.
- Opening individual listings to see full detail and extracted visual characteristics.
- Cross-filtering charts to isolate segments.
- Exporting data.

### System actions

**Shop discovery and selection**
1. Identifies candidate shops selling in the niche and product type.
2. Scores each candidate on estimated sales, estimated revenue, review count, review velocity, active listing count and shop age.
3. **Deliberately favours shops under three years old** — a shop that reached strong sales recently demonstrates a *currently replicable* strategy, whereas a ten-year-old shop's advantage is largely accumulated authority a new entrant cannot copy. Young winners are better teachers.
4. Applies qualification gates (minimum sales or reviews, minimum relevant listings, shop currently active).
5. Selects the top 20. If fewer than 20 qualify, falls back to 10; if fewer than 10, falls back to 5; if fewer than 5, proceeds with what exists and marks the entire run as reduced-confidence with a prominent warning.
6. Records *why* each shop was selected and how it ranked.

**Listing collection**
For every relevant listing in every selected shop, captures: title, description, tags, price, shipping cost and whether shipping is free, number of images, product type, listing age, shop age at time of capture, bestseller status, review count, average rating, review velocity, estimated sales, estimated revenue, favourites, and whether personalisation is offered.

**Visual analysis**
For each listing's primary image, extracts: the colour palette and which palette family it belongs to; the typography style (vintage serif, condensed sans, script, handwritten, slab, display bold, retro, minimal, distressed, or none); the layout archetype (centred stack, badge, arched text, left-aligned, illustration-only, text-over-illustration, split, framed, repeating); the mockup presentation style (flat lay, model lifestyle, ghost mannequin, hanging, folded, plain studio, graphic only, in-situ scene); subject matter; and humour type.

**Textual analysis**
Extracts keyword and tag frequency across all collected listings, weighted against a general baseline so that terms which are merely common everywhere do not appear significant.

### Expected output
- A table of selected shops with their metrics and selection rationale.
- A table of every collected listing with 20+ attributes each.
- Distribution charts: price, image count, review velocity against revenue, shop age against revenue, listing age against sales.
- A complete visual profile for each listing.
- A keyword and tag frequency analysis.

### User decisions
None required — but exploration here often drives the Stage 5 decision.

### Design requirements
- **The fallback ladder must be visible.** If only 10 shops were analysed instead of 20, the user must know, and know why.
- **Every value must show its source and confidence.** Price from the marketplace is measured; monthly sales from a third-party estimate is not, and they must not look the same.
- Tables must handle thousands of rows responsively, with sorting, filtering, column configuration and export.
- Individual listings must be inspectable in full, including the extracted visual analysis, so the user can sanity-check the machine's interpretation.

---

## Stage 5 — Opportunity Report

**Frequency:** Automatic within a run. **The first mandatory human decision gate.**

### User actions
1. Reads the overall opportunity verdict.
2. Examines the five sub-scores and expands any that seem surprising to see the underlying features and reasoning.
3. Reviews the ranked sub-niches.
4. Reads the success factor analysis.
5. Reads the failure factor analysis.
6. Reviews the market gap map.
7. **Decides:** proceed, refine, reject, or deepen.

### System actions

**Opportunity scoring.** Produces an overall 0–100 score composed of Demand, Competition (inverted — less competition scores higher), Trend, Profitability and Seasonality. Maps it to a verdict band: Avoid (0–29), Marginal (30–49), Good (50–69), Strong (70–84), Exceptional (85–100). Every sub-score records its input features, the method used, its confidence, and a plain-English explanation.

**Success analysis.** Divides collected listings into performance cohorts by age-normalised sales, then for every visual, commercial and structural attribute computes: how prevalent it is among top performers, how prevalent it is across the whole market, the ratio between the two (the "lift"), the sample size, and a statistical confidence level.

The output reads as, for example:

> **84%** of top-decile listings use **muted green palettes**
> Market baseline: **40%** · **Lift: 2.1×** · Sample: 42 listings · Confidence: **High**

> **91%** of top-decile listings use **8 or more images**
> Market baseline: **54%** · **Lift: 1.7×** · Sample: 46 listings · Confidence: **High**

> **76%** of top-decile listings use **vintage typography**
> Market baseline: **31%** · **Lift: 2.5×** · Sample: 38 listings · Confidence: **High**

**Critically:** the baseline is always shown. A bare "84% use green" is meaningless if 82% of *all* listings use green. Any finding with too small a sample or insufficient statistical significance is suppressed from the ranked list and shown separately as insufficient evidence.

It also produces a synthesis — "the median winning listing" — as a single specification sheet: modal palette family, modal typography, modal layout, median price, median image count, modal mockup style. This is the artefact users will actually act on.

If the user selected Auto-Select Best Style, the winning style is determined here from measured performance, and the evidence for that choice is shown.

**Failure analysis.** The same statistical treatment applied to persistently under-performing listings, producing an explicit "avoid these" list. Additionally identifies "crowded loser" patterns — things that are simultaneously very common *and* over-represented among failures, i.e. the trap most of the market has fallen into. Distinguishes findings with a plausible causal mechanism (one image, unused tags) from those that are merely correlated, and labels them accordingly.

**Market gap analysis.** Maps sub-niche coverage against sub-niche demand, scores each underserved intersection, and ranks the results. **Applies a demand floor**: a segment with no supply *and* no demand is a desert, not an opportunity, and is excluded rather than recommended. Every returned gap shows both its demand evidence and its supply count so the user can verify this themselves. Flags gaps that exist for a reason — trademark-heavy territory, seasonally dead periods, or ideas that cannot be expressed as a printable graphic.

### Expected output
- Overall opportunity score with a verdict band and a concise executive summary naming the biggest opportunity and the biggest risk.
- Five explained sub-scores.
- Ranked sub-niches, each independently scored.
- Weighted success factors with full statistical context.
- Weighted failure factors, causality-labelled.
- A printable "Do / Avoid" comparison sheet.
- A ranked list of market gaps with a demand-versus-supply visualisation.

### User decisions
**This is the most important decision in the product.**

| Decision | Consequence |
|---|---|
| **Proceed** | Continue to concept generation |
| **Refine** | Start a new run narrowed to a specific promising sub-niche |
| **Reject** | Mark the niche as rejected with a reason, so it appears in a "considered and rejected" list and is never blindly re-researched |
| **Deepen** | Upgrade a Quick or Standard run to Deep, reusing all existing data and fetching only the difference |

### Design requirements
- **A reading order must be enforced by the interface:** Opportunity → Competitors → Success → Failure → Gaps. Each with a clear "next" affordance.
- **Every claim must be one click from its evidence.** Clicking "84% use muted greens" must show the 35 specific listings.
- **Rejection must be a first-class outcome, not a dead end.** A report that says "do not enter this market" has saved the user weeks and should be celebrated as a successful outcome, not treated as a failure.
- Degraded results must be visibly degraded, with a specific statement of what is missing and the one action that would fix it.

---

## Stage 6 — Design Generation (Concepts)

**Frequency:** After proceeding from Stage 5. Target duration: 60–90 seconds.

### User actions
Primarily waiting, then reviewing.

### System actions
1. Generates **10 concepts derived from success factors** — designs engineered to match what demonstrably works in this niche.
2. Generates **10 concepts derived from market gaps** — designs targeting the underserved space.
3. For each concept produces: a name, a description, the target audience, the design angle, the sub-niche it serves, the visual direction (palette family, typography class, layout archetype), any text content, and **reasoning that cites the specific success factors or gaps that produced it**.
4. Scores every concept on five dimensions:
   - **Market Fit** — alignment with proven success factors, minus alignment with known failure patterns.
   - **Originality** — distance from everything already in the market and from the user's own prior work.
   - **Conversion** — likelihood of turning views into purchases, based on attributes empirically associated with conversion.
   - **Competition** — how crowded this specific angle already is.
   - **Opportunity** — the size of the gap or sub-niche it addresses.
5. Combines these into a single **Opportunity Score** with a band (Weak / Moderate / Promising / Strong / Exceptional), and shows exactly which factors contributed positively and negatively.
6. Removes near-duplicates within the set and flags anything closely resembling a concept the user has generated before, regenerating as needed to deliver a full set of twenty genuinely distinct ideas.

**Crucially:** concept generation receives only aggregate statistics — palette families, typography classes, layout archetypes, price bands, gap descriptions. It never receives competitor listing text, titles or images. Originality is therefore structural: there is no path by which a competitor's specific design could influence what is created.

### Expected output
Twenty original concepts, each with a name, description, audience, style direction, reasoning, evidence citations, and six scores.

### User decisions
None yet — this feeds Stage 7.

### Design requirements
- **Concepts must be genuinely distinct.** Twenty variations of one idea is a failure, and the system must detect and correct this automatically.
- **Reasoning must be specific and checkable.** "This design targets composting enthusiasts, an underserved segment (12 competing listings against a demand index of 61), using the muted earth palette that shows 2.1× lift in this niche" — not "this design will appeal to gardeners".
- **Every cited factor must be clickable** through to the evidence behind it.

---

## Stage 7 — Artwork Selection

**Frequency:** After concepts are generated. **The second mandatory human decision gate.**

### User actions

**Concept selection**
1. Reviews the twenty concepts as a sortable, filterable board.
2. Opens individual concepts to read full descriptions, scores and reasoning.
3. Regenerates individual concepts with optional steering, or the whole set.
4. Optionally adds their own concept manually, which then flows through the same scoring and screening.
5. Optionally expands a promising concept into five variants along a chosen axis.
6. **Selects the concepts they want made into artwork** — typically 3–6 of the 20.

**Legal review**
7. Reviews the legal risk assessment for each selected concept.
8. Accepts safer alternatives where risk was found, or acknowledges and overrides medium risk, or abandons blocked concepts.

**Artwork review**
9. Reviews the generated artwork brief and edits it if desired.
10. Triggers generation, then compares the resulting variants side by side.
11. Reviews the print-readiness assessment for each.
12. Applies fixes — background removal, upscaling, cropping, vectorising — or regenerates with steering.
13. Reviews the originality comparison against the market.
14. **Accepts the winning variant.**

### System actions

**Legal and safety screening — runs before any artwork is generated**
- Extracts potentially protected entities from each concept: brand names, character names, celebrity names, sports teams, slogans, media titles.
- Checks these against an internal high-risk blocklist, live trademark registries across multiple jurisdictions in the relevant goods classes, and marketplace-prohibited terms.
- Assesses copyright risk: derivative-work indicators, recognisable characters or likenesses, quoted lyrics or dialogue.
- Assigns a risk level: None, Low, Medium, High, or Blocked — determined by an explicit rule set, not by an opinion.
- **Blocked concepts cannot proceed under any circumstance.** High risk requires an explicit typed acknowledgement and a written justification, both permanently recorded. Medium risk requires acknowledgement.
- For anything above Low, generates safer alternative versions that preserve the commercial intent without the risky element, and automatically re-screens them.

**Artwork generation**
- Composes a detailed brief: subject, composition, explicit palette colours derived from the winning palette family, typography direction (by style class, never by named font), texture and finish, transparent background requirement, exact print dimensions, and a list of things to avoid (no logos, no photorealistic faces, no strokes too thin to print, no gradients that will band, no more than a manageable number of colours).
- Generates four variants by default.
- Removes backgrounds to true transparency with edge refinement.
- Crops to content and upscales to 300 DPI at the actual print size.
- Runs print-readiness checks: effective resolution, genuine transparency, minimum stroke width at print size, minimum text height, colour count, printability of the colours used, edge artefacts, whether near-white ink would disappear on a white garment.
- Produces web previews, thumbnails, and a checkerboard proof showing transparency.
- Compares the artwork against every competitor image in the run and against the user's own prior artwork, flagging anything too similar.
- Where the design suits it, produces a scalable vector version.
- Re-screens the finished artwork visually for anything the brief could not guarantee — unintended logos, recognisable characters, or invented text.

### Expected output
- A legal risk assessment per concept with specific matches and links.
- Four artwork variants per selected concept.
- A print-readiness report per variant naming any failing criterion and its specific remedy.
- An originality assessment.
- Print-ready files plus web previews.

### User decisions
- **Which concepts to make** — the primary creative decision, and the point at which the user's judgement is genuinely required.
- **Whether to accept legal risk**, where any exists.
- **Which artwork variant wins.**
- Whether to accept, regenerate, or revise the brief.

### Design requirements
- **No artwork may be generated for an unscreened or blocked concept, under any circumstances.** This must be impossible at the system level, not merely hidden in the interface.
- **The cost of generation must be shown before it is incurred**, and must update live as concepts are selected.
- **Print-readiness failures must be specific and actionable.** "Effective resolution is 180 DPI at 15×18 inches; 300 DPI required — upscale, or regenerate at higher resolution" — never "QA failed".
- **Artwork that fails print-readiness must not be attachable to a product.**
- The legal disclaimer must be persistent: this reduces risk, it is not legal advice, and it does not guarantee non-infringement.

---

## Stage 8 — Product Creation

**Frequency:** Per accepted artwork. Target duration: 2–3 minutes.

### User actions
1. Reviews recommended product configurations.
2. Selects a blueprint and print provider.
3. Selects colours and sizes to offer.
4. Reviews the cost breakdown and sets a price, either by target margin or directly.
5. Adjusts artwork placement if the default is not right.
6. Reviews and orders the generated mockups.

### System actions
- Ranks available product configurations by Demand (which garment types and colours appear most among successful listings), Competition (how crowded that exact configuration is) and Profitability (real margin after all costs and fees).
- Recommends colours empirically — the colours that actually appear among winners — and validates them against the artwork, warning where dark artwork on a dark garment will not read.
- Computes a full cost breakdown: production cost, shipping, listing fee, transaction fee, payment processing, advertising fees, tax treatment, and resulting net profit and margin.
- **Models advertising fees as charged by default.** Optimistic margin arithmetic is the single most common way POD sellers lose money, and the product refuses to participate in it. Both the with-ads and without-ads figures are shown.
- Enforces the user's minimum margin: configurations below it are shown but flagged and cannot be selected without an explicit override.
- Calculates artwork placement appropriate to the product type.
- Uploads artwork, creates the product, and retrieves mockups.
- Orders mockups by the presentation styles that perform best in this niche.

### Expected output
- A ranked table of product configurations with scores, costs, prices and margins.
- A configured product with selected variants and pricing.
- A gallery of mockups in a recommended order.
- An itemised profit calculation.

### User decisions
- Which blueprint and provider — a genuine trade-off between cost, quality and fulfilment speed.
- Which colours and sizes to offer.
- Price point, within the constraint of the margin floor.
- Whether to offer free shipping, with the price implications recalculated live.

### Design requirements
- **Every cost line must be itemised and visible.** No black-box margin figure.
- **Competitor price context must be shown** — where does this price sit relative to the market's distribution?
- **Margin floor violations must block, not warn**, and must state the exact price required to clear the floor.

---

## Stage 9 — SEO Generation

**Frequency:** Per product. Target duration: 60 seconds.

### User actions
1. Reviews ten generated listing variations.
2. Reads the keyword evidence behind the recommendations.
3. Edits any variation directly, with live validation.
4. Regenerates individual variations or the whole set.
5. **Selects the variation to use.**

### System actions
- Builds a keyword pool from competitor tags weighted by the sales of the listings that used them, cross-referenced against a general baseline so that generically common terms do not dominate, plus sub-niche terms and any user-supplied seeds.
- Generates ten variations, each deliberately positioned along a different axis: gift-focused, audience-focused, humour-led, benefit-led, occasion-led, long-tail-specific, broad-reach, seasonal, personalisation-led, and premium-positioned. Each is labelled with its axis.
- Each variation includes a title, a structured description (hook, product details, materials and care, sizing, shipping, gift angle, keyword section), exactly thirteen tags, ranked keywords with the evidence for each, a positioning statement, and suggested materials.
- Enforces marketplace constraints as hard rules: title length, exactly thirteen tags, tag length, no duplicate tags.
- Scores each variation on keyword coverage, how early the primary keyword appears, tag diversity, readability, and whether the primary keyword's competition level is sensible — deliberately *not* preferring zero-competition keywords, since a keyword nobody searches is not an advantage.
- Screens all generated text for restricted and trademarked terms.

### Expected output
Ten complete, validated, differentiated listing variations, ranked by quality score, each with its keyword evidence.

### User decisions
- Which variation to use.
- Whether to edit — and heavy editing is a signal that the engine needs improvement, tracked as a product metric.

### Design requirements
- **Live validation with visible counters.** The user must never submit a listing that the marketplace will reject.
- **Every keyword must show its evidence:** which competitor listings used it, and what they earned.
- **The nine unselected variations must be retained** for future optimisation experiments.
- A realistic preview showing how the listing will appear in search results and on the listing page.

---

## Stage 10 — Final Approval

**Frequency:** Per product. **The third and final mandatory human decision gate.** Target duration: 60–90 seconds.

### User actions
1. Reviews everything on a single screen.
2. Checks the pre-publish checklist.
3. Fixes anything flagged.
4. **Approves and publishes** — or saves for later.

### System actions
Presents, on one page:
- The concept summary and its Opportunity Score.
- The artwork, with a transparency proof.
- All mockups in publish order, with the primary image indicated.
- The product configuration and variant table.
- The full pricing table and itemised profit estimate.
- The selected listing content, rendered as it will appear.
- The legal screening status, including any overrides and who made them.
- A pre-publish checklist.

**The checklist distinguishes hard requirements from advisories:**

| Check | Type |
|---|---|
| Legal screening cleared (or overridden with a record) | **Hard — blocks publishing** |
| Print-readiness passed | **Hard** |
| Originality check passed or acknowledged | **Hard** |
| Margin at or above the floor | **Hard** |
| Title present and within length limit | **Hard** |
| Exactly thirteen tags, each within length limit | **Hard** |
| Product category and shipping profile set | **Hard** |
| Production partner declared | **Hard** |
| Image count at or above the recommended level | Advisory |
| Price within the competitive range | Advisory |
| Description of adequate length | Advisory |

Publishing is disabled until every hard check passes. On confirmation, the listing goes live, the action is permanently recorded, and the first performance check is scheduled.

### Expected output
A published, live listing — or a saved draft in a review queue.

### User decisions
- **Publish or not.** This is deliberately the only action in the product with a confirmation dialogue restating exactly what will happen and to which shop.
- Whether to override any advisory warnings.

### Design requirements
- **Everything on one screen.** The user must not need to navigate to verify anything.
- **The system must never publish autonomously.** Not on a schedule, not in bulk without confirmation, not ever.
- **Hard checks must be enforced by the system, not merely reflected in a disabled button.**
- The page must be printable, because users will want a record.

---

## Stage 11 — Publishing

**Frequency:** Triggered by Stage 10 approval. Target duration: under 60 seconds.

### User actions
Confirms. Then observes.

### System actions
- Creates the product in the fulfilment system with the correct blueprint, provider, variants, pricing and artwork placement.
- Retrieves and stores mockups.
- Creates the marketplace listing **as a draft first**, with full content, category, shipping profile, policies, and production partner declaration.
- Uploads images in the specified order with the correct primary image.
- Configures variants, per-variant pricing and stock-keeping identifiers.
- Links the fulfilment product to the marketplace listing so that orders route correctly — a step that is mandatory and appears on the checklist, because omitting it means orders will not fulfil.
- Transitions the listing to active only on explicit approval.
- Records the complete published payload permanently.
- Schedules the first performance check.

**Failure handling.** Publishing is broken into independent operations, each of which can succeed or fail on its own. If images fail after the listing is created, the user is offered a targeted retry that uploads only the missing images — it never creates a second listing. Duplicate listings are treated as the most operationally damaging failure the product could cause, and are prevented at multiple levels.

### Expected output
A live listing with a link, a confirmed fulfilment linkage, and a permanent record of exactly what was published.

### User decisions
- Whether to retry a partially failed publish.
- Whether to immediately begin another product from the same research.

### Design requirements
- **Never create a duplicate listing.** Non-negotiable.
- **Partial failures must be repairable without starting over** and without regenerating anything.
- **Authorisation problems must pause work, not destroy it** — a revoked marketplace connection should leave everything resumable after reconnection.

---

## Stage 12 — Tracking Results

**Frequency:** Continuous and automatic.

### User actions
1. Reviews portfolio performance on a dashboard.
2. Examines individual listing performance.
3. Compares predicted scores against actual outcomes.
4. Reviews which niches, sub-niches and concept origins are performing.
5. Reviews and decides on proposed improvements to the system's own scoring.
6. Starts new research informed by what has worked.

### System actions

**Performance tracking**
- Checks each published listing daily — more frequently in the first week after publishing, when the signal is most decision-relevant — for views, favourites, orders, revenue and status.
- Records each check as a permanent historical record rather than overwriting.
- Computes derived measures: views per day, conversion rate, revenue per day, days to first sale, and performance percentile adjusted for how long the listing has been live.
- Detects when a listing has been edited outside the system, or deactivated by the marketplace.

**Analytics**
- Portfolio performance over time with markers showing when products were published.
- Per-niche and per-sub-niche performance comparison.
- **Comparison of success-derived concepts against gap-derived concepts** — a directly actionable finding about which strategy works better for this user.
- Prediction accuracy: how well the Opportunity Score actually predicted outcomes, shown as a calibration curve.
- Cost analysis: spend per research run, per artwork, per published product, against budget.

**Learning**
- Builds a record for each published listing linking its design, listing and pricing attributes to its realised performance.
- Once sufficient outcomes exist, fits improved weightings against real results, using a time-based split so that the test is honest.
- Blends fitted weights toward the original expert-set values in proportion to sample size, so that a run of luck on a dozen listings cannot rewrite the model.
- Tests the proposal against held-back historical data and reports whether it genuinely improves.
- **Presents a proposal to the user. Never activates automatically.** The user sees exactly which weights would change, in which direction, and how the proposal performed against history, and decides.
- Below a minimum number of outcomes, refuses to fit at all and says so plainly.
- Reports which success factors actually turned out to be predictive and which did not — closing the loop visibly.

### Expected output
- Performance dashboards.
- Prediction accuracy analysis.
- Cost analysis.
- Periodic proposals to improve the system's own scoring, with evidence.
- Notifications: first sale, listing deactivated, no views after 14 days, budget threshold reached.

### User decisions
- Which niches to invest more in.
- Whether to accept a proposed scoring improvement.
- Which under-performing listings to revise or remove.

### Design requirements
- **Full traceability.** Every published listing must be traceable back through its product, artwork, concept, the gap or factor that inspired it, and the research run that discovered it — with each step clickable.
- **The system must be honest when its predictions were wrong.** The calibration view exists to show this, and hiding it would undermine the product's entire premise.
- **Improvements must never be applied silently.** The user must always be able to see what changed and revert it.

---

## 5.13 End-to-end timing summary

| Stage | Duration | Attention required |
|---|---|---|
| 1 · Setup | 10 min | Full (one-time) |
| 2 · Niche selection | 90 sec | Full |
| 3 · Market research | 2–4 min | None |
| 4 · Competitor analysis | 3–5 min | None |
| 5 · Opportunity report | 5 min | **Full — key decision** |
| 6 · Concept generation | 90 sec | None |
| 7 · Selection, legal, artwork | 5 min | **Full — key decision** |
| 8 · Product creation | 3 min | Partial |
| 9 · SEO | 1 min | Partial |
| 10 · Final approval | 90 sec | **Full — key decision** |
| 11 · Publishing | 60 sec | None |
| 12 · Tracking | ongoing | Periodic |

**Total elapsed for one product from a fresh niche: ~28 minutes.
Total user attention required: ~15 minutes.
Additional products from the same research: ~8 minutes each.**

Compare with the manual equivalent: 4–6 hours for the research, plus 2–3 hours per product.

---

# 6. Functional Requirements

Requirements are grouped by capability. Each has an identifier, a priority (**Must** / **Should** / **Could**), and states behaviour rather than implementation.

---

## 6.1 Market Research

| ID | Priority | Requirement |
|---|---|---|
| FR-MR-01 | Must | Accept a free-text niche (2–80 characters) and a product type selected from a defined set |
| FR-MR-02 | Must | Match entered niches against previously researched ones and offer reuse rather than silently creating near-duplicates |
| FR-MR-03 | Must | Offer three research depths with clearly stated differences in scope, duration and cost |
| FR-MR-04 | Must | Accept an optional design style preference including an automatic option that lets the market decide |
| FR-MR-05 | Must | Accept optional seed keywords and excluded terms that constrain the research |
| FR-MR-06 | Must | Show an estimated duration and cost before the run starts |
| FR-MR-07 | Must | Discover and rank at least 5 sub-niches, targeting 8–15, each with a description, rationale and example search terms |
| FR-MR-08 | Must | Derive sub-niches from multiple independent signals and record which supported each |
| FR-MR-09 | Must | Estimate demand for the niche and each sub-niche, with evidence |
| FR-MR-10 | Must | Assess competition intensity, accounting for saturation, incumbent entrenchment and review advantage |
| FR-MR-11 | Must | Determine trend direction with the method used clearly stated |
| FR-MR-12 | Must | Calculate profitability from real production costs and the user's own fee assumptions — never from an estimate |
| FR-MR-13 | Must | Produce a twelve-month seasonality profile identifying peaks, troughs and seasonal strength |
| FR-MR-14 | Must | Produce a single 0–100 opportunity score with a plain-language verdict band |
| FR-MR-15 | Must | Explain every sub-score with its inputs, method and confidence level |
| FR-MR-16 | Must | Rank sub-niches by their own independent opportunity scores |
| FR-MR-17 | Must | Continue and produce a labelled partial result when a data source is unavailable, rather than failing |
| FR-MR-18 | Should | Produce a written executive summary naming the biggest opportunity and the biggest risk |
| FR-MR-19 | Should | Show where this niche ranks against all previously researched niches |
| FR-MR-20 | Should | Recommend the best time to publish, derived from seasonality and a configurable lead time |

---

## 6.2 Competitor Analysis

| ID | Priority | Requirement |
|---|---|---|
| FR-CA-01 | Must | Identify shops selling in the target niche and product type |
| FR-CA-02 | Must | Score and rank candidate shops on sales, revenue, reviews, review velocity, listing count and age |
| FR-CA-03 | Must | Apply a deliberate preference for shops under three years old, with the rationale visible to the user |
| FR-CA-04 | Must | Select up to 20 shops, falling back to 10, then 5, then whatever qualifies — marking the run reduced-confidence below 5 |
| FR-CA-05 | Must | Record and display why each shop was selected and how it ranked |
| FR-CA-06 | Must | Collect all relevant listings from each selected shop, within defined limits, reporting any truncation |
| FR-CA-07 | Must | Capture per listing: title, description, tags, price, shipping cost, free-shipping status, image count, product type, listing age, shop age, bestseller status, reviews, rating, review velocity, estimated sales, estimated revenue, favourites, personalisation availability |
| FR-CA-08 | Must | Extract the colour palette from each listing image and classify it into a defined palette family |
| FR-CA-09 | Must | Classify typography style into a defined vocabulary |
| FR-CA-10 | Must | Classify layout archetype into a defined vocabulary |
| FR-CA-11 | Must | Classify mockup presentation style into a defined vocabulary |
| FR-CA-12 | Must | Extract subject matter and humour type |
| FR-CA-13 | Must | Analyse keyword and tag frequency, weighted against a general baseline so generically common terms do not dominate |
| FR-CA-14 | Must | Label every value with its source and confidence, visually distinguishing measured from estimated |
| FR-CA-15 | Must | Retain observations as dated historical records so the same market can be compared over time |
| FR-CA-16 | Must | Allow sorting, filtering, column configuration and export of all collected data |
| FR-CA-17 | Must | Allow inspection of any individual listing including its extracted visual analysis |
| FR-CA-18 | Should | Present distribution charts for price, image count, review velocity, shop age and listing age |
| FR-CA-19 | Should | Reuse recently collected data rather than re-fetching, to reduce cost and time |

---

## 6.3 Success Analysis Engine

| ID | Priority | Requirement |
|---|---|---|
| FR-SA-01 | Must | Divide listings into performance cohorts using sales normalised for how long each listing has been live |
| FR-SA-02 | Must | For every attribute, compute prevalence among top performers, prevalence across the whole market, the ratio between them, sample size, and statistical confidence |
| FR-SA-03 | Must | **Always display the market baseline alongside the cohort figure**, so that lift is visible and a bare percentage can never mislead |
| FR-SA-04 | Must | Suppress findings with insufficient sample size or statistical significance from the ranked list, showing them separately as insufficient evidence |
| FR-SA-05 | Must | Analyse colours and palette families |
| FR-SA-06 | Must | Analyse typography styles |
| FR-SA-07 | Must | Analyse layout archetypes |
| FR-SA-08 | Must | Analyse product presentation and mockup styles |
| FR-SA-09 | Must | Analyse pricing, identifying the price band associated with the best performance |
| FR-SA-10 | Must | Analyse image counts, identifying the optimal range |
| FR-SA-11 | Must | Analyse keywords and tags used by top performers |
| FR-SA-12 | Must | Analyse listing structure — title length, tag utilisation, description characteristics |
| FR-SA-13 | Must | Assign each finding a weight reflecting its strength, confidence and sample size, for use in later scoring |
| FR-SA-14 | Must | Produce a "median winning listing" synthesis as a single specification sheet |
| FR-SA-15 | Must | When automatic style selection was chosen, determine the winning style from measured performance and show the evidence |
| FR-SA-16 | Should | Identify correlations between attributes and flag where attributes are measuring the same thing |
| FR-SA-17 | Should | Identify interaction effects where two attributes together outperform their individual contributions |
| FR-SA-18 | Should | Present visual galleries of winning palettes, typography and layouts |

---

## 6.4 Failure Analysis Engine

| ID | Priority | Requirement |
|---|---|---|
| FR-FA-01 | Must | Identify persistently under-performing listings, excluding those too new to judge |
| FR-FA-02 | Must | Apply the same statistical rigour as the success analysis, including baselines and confidence |
| FR-FA-03 | Must | Analyse the same attribute set as the success engine |
| FR-FA-04 | Must | Additionally analyse listing-quality signals: description quality, unused tag slots, keyword stuffing, single-image listings, missing attributes, price outliers |
| FR-FA-05 | Must | Produce paired "success factors" and "avoid these factors" outputs |
| FR-FA-06 | Must | Handle attributes appearing in both lists explicitly, showing the net effect and marking them ambiguous |
| FR-FA-07 | Must | Distinguish findings with a plausible causal explanation from those that are merely correlated, and label them |
| FR-FA-08 | Must | Feed failure findings into concept generation as constraints and into design scoring as penalties |
| FR-FA-09 | Should | Identify "crowded loser" patterns that are both very common and over-represented among failures |
| FR-FA-10 | Should | Produce a printable side-by-side "Do / Avoid" sheet |

---

## 6.5 Market Gap Engine

| ID | Priority | Requirement |
|---|---|---|
| FR-GA-01 | Must | Map coverage across combinations of sub-niche, design angle and style |
| FR-GA-02 | Must | Estimate demand for each sub-niche independently of supply |
| FR-GA-03 | Must | Score each underserved combination on demand, inverse supply, monetisability and feasibility |
| FR-GA-04 | Must | **Apply a demand floor that excludes segments with no meaningful demand.** A market with no supply *and* no demand is a desert, not an opportunity, and must never be presented as one |
| FR-GA-05 | Must | Rank gaps and return the strongest, each with an explanation |
| FR-GA-06 | Must | Display both demand evidence and supply count for every gap, so the user can verify the reasoning |
| FR-GA-07 | Must | Present a demand-versus-supply visualisation with the demand floor drawn and explained |
| FR-GA-08 | Should | Suggest concrete design angles for each top gap |
| FR-GA-09 | Should | Flag gaps that exist for a reason — trademark-heavy territory, seasonally dead periods, or ideas that cannot be printed |
| FR-GA-10 | Should | Present a coverage heat map showing where the market is and is not served |

---

## 6.6 AI Design System

| ID | Priority | Requirement |
|---|---|---|
| FR-DS-01 | Must | Generate exactly 10 concepts derived from success factors |
| FR-DS-02 | Must | Generate exactly 10 concepts derived from market gaps |
| FR-DS-03 | Must | Include for each concept: name, description, target audience, design angle, sub-niche, style, visual direction, text content, and reasoning |
| FR-DS-04 | Must | Cite the specific success factors or gaps behind each concept, with those citations navigable to the underlying evidence |
| FR-DS-05 | Must | **Never reference, describe, imitate or derive from any specific competitor listing, shop, brand or artwork.** Only aggregate statistical characteristics may inform generation |
| FR-DS-06 | Must | Detect and eliminate near-duplicate concepts within a set, regenerating to maintain a full set of distinct ideas |
| FR-DS-07 | Must | Flag concepts closely resembling the user's own previous work |
| FR-DS-08 | Must | Require explicit user selection before any artwork is produced |
| FR-DS-09 | Must | Support regenerating a single concept or the whole set, with optional steering |
| FR-DS-10 | Must | Support the required styles: vintage, typography, hand-drawn, illustration, humour, modern |
| FR-DS-11 | Should | Support manually entered concepts, which then flow through identical scoring and screening |
| FR-DS-12 | Should | Support expanding a promising concept into variants along a chosen axis |

---

## 6.7 Design Success Prediction

| ID | Priority | Requirement |
|---|---|---|
| FR-DP-01 | Must | Score every concept on Market Fit, Originality, Conversion, Competition and Opportunity |
| FR-DP-02 | Must | Combine these into a single Opportunity Score with a plain-language band |
| FR-DP-03 | Must | Show precisely which factors contributed positively and negatively to each dimension |
| FR-DP-04 | Must | Base Market Fit on measured alignment with success factors, less alignment with failure factors |
| FR-DP-05 | Must | Base Originality on measured distance from the existing market and from the user's own prior work |
| FR-DP-06 | Must | Re-score after artwork exists, since originality and conversion signals change once the design is visible |
| FR-DP-07 | Must | Retain every prediction so it can later be compared against the actual outcome |
| FR-DP-08 | Must | **Ensure scores are reproducible** — identical inputs must always produce identical scores |
| FR-DP-09 | Should | Provide a view comparing predicted scores against realised outcomes once data exists |

---

## 6.8 Legal and Safety

| ID | Priority | Requirement |
|---|---|---|
| FR-LS-01 | Must | **Run before any artwork is generated. Generation for an unscreened or blocked concept must be impossible at the system level, not merely hidden in the interface** |
| FR-LS-02 | Must | Extract potentially protected entities: brands, characters, celebrities, teams, slogans, media titles |
| FR-LS-03 | Must | Check against an internal high-risk term list |
| FR-LS-04 | Must | Check against live trademark registries in the relevant jurisdictions and goods classes |
| FR-LS-05 | Must | Check against marketplace prohibited and restricted terms |
| FR-LS-06 | Must | Assess copyright risk including derivative works, recognisable likenesses and quoted material |
| FR-LS-07 | Must | Assign a risk level of None, Low, Medium, High or Blocked, determined by explicit rules |
| FR-LS-08 | Must | Prevent blocked concepts from proceeding under any circumstances |
| FR-LS-09 | Must | Require explicit acknowledgement for medium risk, and a typed confirmation with written justification for high risk |
| FR-LS-10 | Must | Permanently record every screening, every match found, and every override with its author and justification |
| FR-LS-11 | Must | Generate safer alternatives for flagged concepts and re-screen them automatically |
| FR-LS-12 | Must | Screen generated artwork visually for anything the brief could not guarantee |
| FR-LS-13 | Must | Screen final listing text before publishing |
| FR-LS-14 | Must | Display a persistent disclaimer that this reduces risk but is not legal advice |
| FR-LS-15 | Should | Maintain a permanent block list of terms that previously caused a takedown |

---

## 6.9 Artwork Generation

| ID | Priority | Requirement |
|---|---|---|
| FR-AG-01 | Must | Produce a detailed artwork brief specifying subject, composition, exact palette colours, typography direction, texture, background requirement, print dimensions and things to avoid |
| FR-AG-02 | Must | Derive palettes from the winning palette family measured in the research, not from arbitrary choice |
| FR-AG-03 | Must | Allow the user to review and edit the brief before generation |
| FR-AG-04 | Must | Generate multiple variants per concept, with a configurable count |
| FR-AG-05 | Must | Produce artwork with a genuinely transparent background |
| FR-AG-06 | Must | Upscale artwork to meet print resolution requirements at the true print size |
| FR-AG-07 | Must | Validate print readiness: resolution, transparency, minimum stroke width, minimum text size, colour count, printable colours, edge quality, visibility on the intended garment colour |
| FR-AG-08 | Must | **Name the specific failing criterion and its specific remedy** when validation fails |
| FR-AG-09 | Must | Prevent artwork that fails validation from being attached to a product |
| FR-AG-10 | Must | Compare generated artwork against every competitor image in the research and flag anything too similar, requiring acknowledgement |
| FR-AG-11 | Must | **Never use a competitor image as an input to generation, under any circumstances** |
| FR-AG-12 | Must | Produce print files, web previews, thumbnails and a transparency proof |
| FR-AG-13 | Must | Support the required styles: typography-led, vintage, hand-drawn, illustration, modern, humour |
| FR-AG-14 | Must | Support regeneration with steering, within retry limits and budget |
| FR-AG-15 | Should | Produce scalable vector versions where the design suits it |
| FR-AG-16 | Should | Support user-uploaded artwork entering at this stage, subject to identical validation and screening |

---

## 6.10 Product Recommendation and Pricing

| ID | Priority | Requirement |
|---|---|---|
| FR-PR-01 | Must | Maintain a current catalogue of available products, providers, variants and production costs |
| FR-PR-02 | Must | Rank product configurations on demand, competition and profitability |
| FR-PR-03 | Must | Recommend garment colours empirically, from what appears among successful listings |
| FR-PR-04 | Must | Validate colour recommendations against the artwork and warn where the design will not read |
| FR-PR-05 | Must | Itemise the full cost breakdown: production, shipping, listing fee, transaction fee, payment processing, advertising fee, tax |
| FR-PR-06 | Must | **Model advertising fees as charged by default**, showing both the with-ads and without-ads outcome |
| FR-PR-07 | Must | Enforce a user-configured minimum margin, blocking selection below it and stating the price required to clear it |
| FR-PR-08 | Must | Show where the chosen price sits within the competitive price distribution |
| FR-PR-09 | Must | Support T-Shirts, Sweatshirts, Hoodies, Mugs, Posters and Tote Bags |
| FR-PR-10 | Should | Recommend a free-shipping strategy with the price implications calculated |
| FR-PR-11 | Should | Recommend how many listing images to produce, based on the measured optimum |
| FR-PR-12 | Should | Alert when a production cost change affects an unpublished product's margin |

---

## 6.11 SEO System

| ID | Priority | Requirement |
|---|---|---|
| FR-SE-01 | Must | Generate exactly 10 listing variations per product |
| FR-SE-02 | Must | Include per variation: title, structured description, exactly 13 tags, ranked keywords with evidence, and a positioning statement |
| FR-SE-03 | Must | Select keywords based on measured competitor performance, weighted by the sales of the listings that used them |
| FR-SE-04 | Must | Show the evidence behind every keyword recommendation |
| FR-SE-05 | Must | Enforce marketplace constraints as hard rules, with live validation and visible counters |
| FR-SE-06 | Must | Differentiate the ten variations along declared, labelled positioning axes |
| FR-SE-07 | Must | Score and rank variations on keyword coverage, keyword placement, tag diversity, readability and competitive sensibility |
| FR-SE-08 | Must | Support regenerating individual variations or the whole set |
| FR-SE-09 | Must | Support full manual editing with live validation |
| FR-SE-10 | Must | Screen all generated text for restricted and trademarked terms before publishing |
| FR-SE-11 | Must | Retain unselected variations for later optimisation |
| FR-SE-12 | Should | Detect and avoid keyword stuffing and machine-sounding text |
| FR-SE-13 | Should | Suggest the appropriate marketplace category and attributes |
| FR-SE-14 | Should | Show a realistic preview of the listing as it will appear |

---

## 6.12 Publishing System

| ID | Priority | Requirement |
|---|---|---|
| FR-PB-01 | Must | Upload artwork and create the product in the fulfilment system with the chosen configuration |
| FR-PB-02 | Must | Calculate appropriate artwork placement per product type, with manual adjustment available |
| FR-PB-03 | Must | Retrieve, store and order mockups |
| FR-PB-04 | Must | **Create marketplace listings as drafts. Never activate a listing without explicit user approval** |
| FR-PB-05 | Must | Set all listing content, category, shipping, policies and production partner declaration |
| FR-PB-06 | Must | Upload images in the specified order with the correct primary image |
| FR-PB-07 | Must | Configure variants and per-variant pricing |
| FR-PB-08 | Must | Link the fulfilment product to the marketplace listing so orders route correctly, and verify this on the checklist |
| FR-PB-09 | Must | Present a single review screen containing everything needed to make the publish decision |
| FR-PB-10 | Must | Enforce a pre-publish checklist distinguishing hard requirements from advisories, with hard requirements blocking |
| FR-PB-11 | Must | Require explicit confirmation restating the action and its target |
| FR-PB-12 | Must | **Never create duplicate listings**, even under retry, timeout or crash conditions |
| FR-PB-13 | Must | Support targeted retry of partially failed publishing without recreating anything already created |
| FR-PB-14 | Must | Preserve all work when authorisation is lost, and resume after reconnection |
| FR-PB-15 | Must | Permanently record what was published, when, by whom |
| FR-PB-16 | Should | Support a review queue for drafts awaiting approval |
| FR-PB-17 | Could | Support batch publishing with per-item checklists and explicit confirmation |

---

## 6.13 Analytics System

| ID | Priority | Requirement |
|---|---|---|
| FR-AN-01 | Must | Permanently retain all research runs, reports, concepts, artwork, products and listings |
| FR-AN-02 | Must | Track published listing performance: views, favourites, orders, revenue and status |
| FR-AN-03 | Must | Retain performance as dated historical records rather than overwriting |
| FR-AN-04 | Must | Compute derived measures including conversion rate, days to first sale, and age-adjusted performance percentile |
| FR-AN-05 | Must | Provide portfolio and per-niche performance views |
| FR-AN-06 | Must | Compare success-derived against gap-derived concepts |
| FR-AN-07 | Must | Compare estimated Opportunity Scores against actual outcomes |
| FR-AN-08 | Must | Report spending by activity and provider against budget |
| FR-AN-09 | Must | **Provide full traceability** from any published listing back through product, artwork, concept, originating factor or gap, and research run — each step navigable |
| FR-AN-10 | Must | Provide run history with filtering, re-running and comparison between runs of the same niche |
| FR-AN-11 | Should | Detect and notify on first sale, listing deactivation, and zero views after a defined period |
| FR-AN-12 | Should | Export reports and data in standard formats |

---

## 6.14 Learning System

| ID | Priority | Requirement |
|---|---|---|
| FR-LE-01 | Must | Maintain a record linking each published listing's attributes to its realised performance |
| FR-LE-02 | Must | Periodically evaluate whether improved scoring weightings can be derived from actual outcomes |
| FR-LE-03 | Must | Test any proposed improvement against held-back historical data using a time-based split |
| FR-LE-04 | Must | Blend proposed weightings toward the original values in proportion to sample size, preventing small samples from causing large swings |
| FR-LE-05 | Must | **Present improvements as proposals requiring explicit user approval. Never activate automatically** |
| FR-LE-06 | Must | Show precisely what would change and how the proposal performed against history |
| FR-LE-07 | Must | Keep every previous configuration and allow reverting at any time |
| FR-LE-08 | Must | **Never alter historical scores.** Re-scoring produces new records alongside the originals |
| FR-LE-09 | Must | Refuse to fit below a minimum number of outcomes, and state this plainly in the interface |
| FR-LE-10 | Should | Report which success factors turned out to be predictive and which did not |

---

## 6.15 Cross-cutting

| ID | Priority | Requirement |
|---|---|---|
| FR-XC-01 | Must | Show the cost of any spending action before it is taken |
| FR-XC-02 | Must | Enforce per-run and monthly spending limits, pausing rather than overspending |
| FR-XC-03 | Must | Allow cancelling any long-running operation, stopping spend while preserving completed work |
| FR-XC-04 | Must | Resume a failed operation from the point of failure without repeating successful work |
| FR-XC-05 | Must | Stream live progress with named steps, not an opaque progress bar |
| FR-XC-06 | Must | Label every estimated value as an estimate, with its source and confidence |
| FR-XC-07 | Must | Make degraded results visibly degraded, naming what is missing and what would fix it |
| FR-XC-08 | Must | Require confirmation for destructive actions, with a recovery window |
| FR-XC-09 | Must | Provide search across all research, concepts, artwork and listings |
| FR-XC-10 | Must | Display integration health: what is connected, quota consumed, last successful use, last error |
| FR-XC-11 | Should | Notify on run completion, run failure, budget thresholds, legal blocks, and publishing outcomes |
| FR-XC-12 | Should | Export all data in portable formats |

---

# 7. Non-Functional Requirements

Stated as measurable budgets. A requirement without a number is an opinion.

## 7.1 Performance

| Area | Requirement |
|---|---|
| Interactive response | Common interactions complete within 200 ms; page loads within 2.5 s at the 95th percentile |
| Perceived progress | Meaningful partial output visible within 20 seconds of starting a research run |
| Full research (Standard depth) | Complete within 11 minutes at the 95th percentile |
| Opportunity report | Available within 4 minutes |
| Concept generation | 20 concepts within 90 seconds |
| Artwork generation | 4 variants, fully processed, within 4 minutes |
| Publishing | Draft created within 60 seconds |
| Large data views | Tables of several thousand rows remain responsive with sorting and filtering |

## 7.2 Reliability

| Area | Requirement |
|---|---|
| Availability | 99.5% monthly for the single-user product; 99.9% for a future SaaS |
| Run completion | At least 97% of runs complete without manual intervention |
| Data loss | No loss of research, concepts, artwork or listings; at most 5 minutes of data at risk in a disaster |
| Recovery time | Full service restored within 2 hours of a serious failure |
| **Duplicate side effects** | **Zero.** No duplicate listings, no duplicate products, no double charges — under any failure condition |
| Partial success | A failure in one part of a run must never discard the rest |
| Resumability | Every operation must be resumable from its point of failure |

## 7.3 Cost

| Area | Requirement |
|---|---|
| Standard research run | Under £1.20 |
| Deep research run | Under £2.60 |
| Artwork per accepted design | Under £0.60 including retries |
| Fully published product | Under £1.80 in marginal cost |
| Monthly infrastructure | Under £140 for the single-user product |
| Budget enforcement | Spending limits are enforced *before* money is spent, never discovered afterwards |

## 7.4 Accuracy and honesty

| Area | Requirement |
|---|---|
| Reproducibility | Identical inputs must always produce identical scores |
| Statistical suppression | Findings below defined sample-size and significance thresholds must never appear in ranked results |
| Baseline disclosure | Every prevalence figure must be accompanied by the market baseline |
| Estimate labelling | Estimated values must be visually distinct from measured values everywhere they appear |
| Confidence propagation | Low-confidence inputs must produce visibly low-confidence outputs |
| Prediction honesty | The system must report where its own predictions were wrong |

## 7.5 Usability

| Area | Requirement |
|---|---|
| Learnability | A competent Etsy seller completes their first research run without documentation |
| Explanation | Every score is one click from the evidence that produced it |
| Error messages | Every error states what happened, why, and what to do next. Raw technical errors are never shown |
| Progress | Every operation over 3 seconds shows named, determinate progress |
| Cost transparency | Every spending action shows its cost before it occurs |
| Accessibility | Meets WCAG 2.2 AA: keyboard navigable, sufficient contrast, screen-reader compatible, colour never the sole carrier of meaning |
| Screen support | Fully functional at 1280×800 and above; usable on tablet; native mobile out of scope |
| Theme | Light and dark, both meeting contrast requirements |

## 7.6 Security and privacy

| Area | Requirement |
|---|---|
| Authentication | Strong password requirements plus a mandatory second factor |
| Credentials | Marketplace and service credentials encrypted at rest and never displayed in full |
| Authorisation | Sensitive actions — publishing, credential changes, legal overrides, budget increases — require re-authentication |
| Audit | Publishing, legal overrides, credential changes and configuration changes are permanently and immutably recorded |
| Data minimisation | Only public commercial data is collected. **No personal data of buyers, reviewers or shop owners beyond a public shop name** |
| Data usage | User data is never used to train third-party models |
| Retention | Raw competitor observations retained 180 days; derived analysis retained indefinitely; audit records retained 7 years |
| Portability | All data exportable in open formats |
| Deletion | Deletion honoured, with a recovery window before permanent removal |

## 7.7 Compliance and ethics

| Area | Requirement |
|---|---|
| Marketplace policies | All listings correctly declare production method and production partner. The system never messages buyers, manipulates reviews, or interacts with competitor listings |
| Data acquisition | Only compliant methods. The default and always-available path requires no automated access to any third-party system. **No technique whose purpose is to avoid detection will be implemented** — if access requires evasion, the capability is simply unavailable |
| Intellectual property | Competitor data is used for aggregate statistical analysis only, never to reproduce or derive artwork. Legal screening is mandatory before generation |
| Transparency | Every AI-generated artefact is identifiable as such, with its full provenance retained |

## 7.8 Maintainability and extensibility

| Area | Requirement |
|---|---|
| Provider independence | Every external service must be replaceable without redesigning the product |
| Data source independence | The product must remain fully functional using only public marketplace data plus user-supplied files |
| Marketplace extensibility | Adding a second marketplace must not require redesigning the research and analysis capabilities |
| Multi-user readiness | Nothing in the design may obstruct a future conversion to multiple users, teams and subscriptions |
| Configuration over code | Fee assumptions, scoring weights, thresholds and limits must be adjustable without code changes |

## 7.9 Operational

| Area | Requirement |
|---|---|
| Deployment | Updates deploy without downtime and without interrupting running work |
| Backup | Continuous backup with monthly verified restore tests |
| Monitoring | Failures, cost anomalies, quota exhaustion and integration problems raise alerts |
| Diagnosability | Every user-visible error carries a reference that ties it to the underlying record |

---

# 8. User Stories

Format: *As a [role], I want [capability], so that [outcome].* Each carries an identifier, a priority, and acceptance criteria.

## 8.1 Setup and configuration

**US-01 · Connect my shop · Critical**
As a POD seller, I want to connect my Etsy shop securely, so that the system can create listings on my behalf without me sharing my password.
*Accepts:* Authorisation completes in under 2 minutes · only the minimum necessary permissions are requested · connection status and expiry are visible · a failure explains exactly what went wrong.

**US-02 · Connect fulfilment · Critical**
As a POD seller, I want to connect my Printify account, so that products can be created and mockups retrieved automatically.
*Accepts:* Token is validated with a live call · available shops are listed for selection · the product catalogue loads in the background · an account with no connected shop produces a specific, actionable message.

**US-03 · Set my economics · Critical**
As a POD seller, I want to configure my fee assumptions and minimum acceptable margin, so that every profit calculation reflects my actual business.
*Accepts:* All fee components are editable · the margin floor is enforced everywhere · changes recalculate existing unpublished products.

**US-04 · Control my spending · Critical**
As a POD seller, I want to set spending limits, so that I can never be surprised by a bill.
*Accepts:* Per-run and monthly limits configurable · spending is shown live against the limit · reaching the limit pauses work rather than overspending · resuming requires an explicit decision.

**US-05 · Use it before connecting everything · Important**
As a new user, I want to explore research features before connecting my shop, so that I can evaluate the product before committing.
*Accepts:* Research runs fully without publishing connections · publishing features are visibly unavailable with the reason stated · connections can be added later without losing work.

---

## 8.2 Market research

**US-06 · Evaluate a niche quickly · Critical**
As an Etsy seller, I want a clear verdict on whether a niche is worth entering, so that I stop wasting weeks on unviable markets.
*Accepts:* A single 0–100 score with a plain verdict · five explained sub-scores · a written summary naming the biggest opportunity and risk · delivered in under 5 minutes.

**US-07 · Understand the verdict · Critical**
As an Etsy seller, I want to see why a niche scored as it did, so that I can judge whether I agree.
*Accepts:* Every sub-score expands to show its inputs, method and confidence · every figure is traceable to its source · estimates are visibly distinguished from measured values.

**US-08 · Find sub-niches · Critical**
As an Etsy seller, I want the broad niche broken into specific segments, so that I can target a defined audience rather than a vague category.
*Accepts:* At least 5 sub-niches, ideally 8–15 · each with a description, rationale, example search terms and its own score · ranked · each can be researched independently.

**US-09 · Know the season · Important**
As an Etsy seller, I want to know when demand peaks, so that I publish with enough lead time to rank before the peak.
*Accepts:* Twelve-month profile · peaks and troughs identified · seasonal strength quantified · a recommended publishing window.

**US-10 · Choose research depth · Important**
As an Etsy seller, I want to control how thorough the research is, so that I can trade cost and time against confidence.
*Accepts:* Three options with concrete stated differences · estimated time and cost per option · a Quick run can later be upgraded without repeating work.

**US-11 · Not repeat myself · Important**
As an Etsy seller, I want to be told when I have already researched a niche, so that I do not pay twice for the same answer.
*Accepts:* Near-matches are detected on entry · prior date and result are shown · the choice to view, refresh or start fresh is offered.

**US-12 · Record rejections · Important**
As an Etsy seller, I want to record that I rejected a niche and why, so that I do not re-investigate it in six months having forgotten.
*Accepts:* Rejection with a reason is a first-class action · rejected niches are listed separately · re-entering one surfaces the prior decision.

---

## 8.3 Competitor analysis

**US-13 · See who is winning · Critical**
As an Etsy seller, I want to see the successful shops in a niche with their real numbers, so that I know who I am actually competing with.
*Accepts:* Up to 20 shops with sales, revenue, reviews, velocity, listing count and age · selection rationale shown per shop · all values labelled by source and confidence.

**US-14 · Learn from replicable success · Important**
As an Etsy seller, I want the analysis to favour shops that succeeded recently, so that I learn strategies I can actually copy rather than accumulated authority I cannot.
*Accepts:* Shops under three years old are preferred · the preference is visible and explained · shop age is shown in every view.

**US-15 · Study every listing · Critical**
As an Etsy seller, I want every relevant listing from those shops in one sortable table, so that I can find patterns myself as well as reading the system's conclusions.
*Accepts:* 20+ attributes per listing · sortable, filterable, configurable, exportable · thousands of rows remain responsive · individual listings open in full detail.

**US-16 · See what they look like · Critical**
As an Etsy seller, I want the visual characteristics of competitor designs extracted into data, so that I can analyse aesthetics rather than merely browse them.
*Accepts:* Palette, palette family, typography style, layout archetype and mockup style per listing · classifications inspectable and sanity-checkable · used consistently across all analysis.

**US-17 · Know when data is thin · Critical**
As an Etsy seller, I want to be told clearly when there was not enough data, so that I do not act on a weak conclusion believing it is strong.
*Accepts:* Fewer than five qualifying shops marks the run reduced-confidence · the warning is prominent · affected conclusions are individually flagged.

---

## 8.4 Success and failure analysis

**US-18 · Know what works here · Critical**
As an Etsy seller, I want to know what successful listings in this specific niche have in common, so that my design decisions are informed rather than instinctive.
*Accepts:* Ranked findings covering colour, typography, layout, presentation, pricing, images and keywords · each shows prevalence among winners, market baseline, ratio, sample size and confidence.

**US-19 · Not be fooled by a big percentage · Critical**
As an Etsy seller, I want to see the market baseline alongside every finding, so that I can tell a real pattern from something merely common.
*Accepts:* Baseline is always displayed · the ratio is always displayed · a finding cannot be presented without both.

**US-20 · Not be fooled by a small sample · Critical**
As an Etsy seller, I want findings based on too little data to be excluded or clearly marked, so that I am not misled by coincidence.
*Accepts:* Findings below the threshold are excluded from the ranked list · they are shown separately as insufficient evidence · thresholds are stated.

**US-21 · Get one clear summary · Important**
As an Etsy seller, I want a single summary of what the typical winning listing looks like, so that I have one artefact to design against.
*Accepts:* One card covering palette, typography, layout, price, image count and presentation · directly usable as a design brief.

**US-22 · Know what fails · Critical**
As an Etsy seller, I want to know what under-performing listings have in common, so that I stop repeating the market's mistakes.
*Accepts:* Same statistical rigour as the success analysis · covers listing-quality issues as well as design · produces an explicit "avoid" list · a printable Do/Avoid sheet.

**US-23 · Know what is guessing · Important**
As an Etsy seller, I want to know which failure findings have a real explanation and which are only correlated, so that I weigh them appropriately.
*Accepts:* Each finding labelled as causally plausible or correlation-only · the distinction explained · findings appearing in both lists shown with their net effect.

**US-24 · Avoid the common trap · Nice-to-have**
As an Etsy seller, I want to know which very common practices are actually associated with failure, so that I stop copying the majority.
*Accepts:* Patterns that are both widespread and over-represented in failures are identified and explained.

**US-25 · Let the data pick the style · Important**
As an Etsy seller, I want the system to determine the best-performing design style rather than relying on my taste.
*Accepts:* Automatic selection is available and recommended · the winning style is determined from measured performance · the evidence and runners-up are shown.

---

## 8.5 Market gaps

**US-26 · Find the empty space · Critical**
As an Etsy seller, I want to find underserved segments, so that I can compete where there is room rather than head-on.
*Accepts:* Ranked gaps with scores · demand evidence and supply count shown for each · a demand-versus-supply visualisation · suggested design angles.

**US-27 · Not be sent into a desert · Critical**
As an Etsy seller, I want the system to never recommend an empty market that is empty because nobody wants it.
*Accepts:* A demand floor is applied and visible · segments below it are excluded, not ranked low · the floor is drawn on the visualisation with an explanation.

**US-28 · Know why a gap exists · Important**
As an Etsy seller, I want to be warned when a gap exists for a bad reason, so that I do not walk into a trademark minefield or a dead season.
*Accepts:* Gaps are flagged for trademark risk, seasonal deadness or physical impracticality, with the reason stated.

---

## 8.6 Design generation

**US-29 · Get ideas grounded in evidence · Critical**
As an Etsy seller, I want design concepts derived from what actually works here, so that my creative direction is informed by the market rather than by my mood.
*Accepts:* 10 concepts from success factors and 10 from gaps · each with name, description, audience, style direction and reasoning · reasoning cites specific findings.

**US-30 · Check the reasoning · Critical**
As an Etsy seller, I want to click through from a concept's reasoning to the evidence behind it, so that I can verify the logic rather than trust it.
*Accepts:* Every cited factor or gap is navigable · the underlying statistic is shown · it matches what the reasoning claims.

**US-31 · Get genuinely different ideas · Important**
As an Etsy seller, I want twenty distinct concepts rather than twenty rewordings of one, so that I have a real choice.
*Accepts:* Near-duplicates are automatically detected and replaced · concepts resembling my own prior work are flagged · the final set is verifiably varied.

**US-32 · Know which ideas are strongest · Critical**
As an Etsy seller, I want each concept scored on its likelihood of succeeding, so that I can prioritise where to spend my creative effort.
*Accepts:* Five dimensions plus a combined score with a plain band · positive and negative contributions itemised · identical inputs always produce identical scores.

**US-33 · Steer the output · Important**
As an Etsy seller, I want to regenerate concepts with direction, so that I can push the results toward what I want without starting over.
*Accepts:* Individual and bulk regeneration · optional steering text · unaffected concepts preserved · cost shown before regenerating.

**US-34 · Add my own idea · Nice-to-have**
As an Etsy seller, I want to enter my own concept, so that my intuition can be tested by the same scoring the system applies to its own ideas.
*Accepts:* Manual concepts receive identical scoring and screening · they appear alongside generated ones.

**US-35 · Explore a promising direction · Nice-to-have**
As an Etsy seller, I want to generate variations of a concept I like, so that I can explore a direction properly before committing.
*Accepts:* Expansion produces several variants along a chosen axis · each is independently scored.

---

## 8.7 Legal safety

**US-36 · Not get my shop suspended · Critical**
As an Etsy seller, I want every concept checked for trademark and copyright risk before any artwork is made, so that I never build a product I cannot legally sell.
*Accepts:* Screening runs before generation, always · checks multiple registries in relevant classes · assigns a clear risk level · blocked concepts cannot proceed under any circumstance.

**US-37 · Understand the risk · Critical**
As an Etsy seller, I want to see exactly what was flagged and why, so that I can make an informed judgement rather than obey an opaque refusal.
*Accepts:* Matched terms shown · each matched registration shown with owner, number, class, jurisdiction and a link · rationale stated · disclaimer present.

**US-38 · Get a safe alternative · Important**
As an Etsy seller, I want safer versions of flagged concepts, so that a good commercial idea is not simply lost to a legal problem.
*Accepts:* Alternatives are generated automatically · they preserve the commercial intent · they are re-screened · they can be accepted in place of the original.

**US-39 · Accept risk deliberately · Important**
As an Etsy seller, I want to be able to override a medium-risk flag with full awareness, so that I retain control while creating an accountable record.
*Accepts:* Medium risk requires acknowledgement · high risk requires typed confirmation and written justification · overrides are permanently recorded · blocked can never be overridden.

---

## 8.8 Artwork

**US-40 · Get print-ready artwork · Critical**
As an Etsy seller, I want artwork that is genuinely ready to print, so that I stop spending 40 minutes per design fixing files.
*Accepts:* Transparent background · 300 DPI at the true print size · validated against every print requirement · downloadable in print and web formats.

**US-41 · Know exactly what is wrong · Critical**
As an Etsy seller, I want validation failures to name the specific problem and the specific fix, so that I am not left guessing.
*Accepts:* Each criterion reports pass, warn or fail with the measured value and the threshold · each failure names its remedy · one-click fixes where possible.

**US-42 · Compare options · Important**
As an Etsy seller, I want several variants per concept side by side, so that I can choose the best rather than accepting the first.
*Accepts:* Multiple variants generated · comparable side by side with zoom · transparency proof toggle · garment-colour preview.

**US-43 · Not accidentally copy anyone · Critical**
As an Etsy seller, I want my artwork compared against the competitors analysed, so that I have confidence it is genuinely original.
*Accepts:* Every artwork compared against every competitor image in the run and my own prior work · anything too similar is flagged with a side-by-side comparison and requires acknowledgement · competitor images are never used as generation input.

**US-44 · Direct the artwork · Important**
As an Etsy seller, I want to see and edit the brief before generation, so that I have creative control rather than pot luck.
*Accepts:* Brief is visible and editable before generating · edits are preserved · regeneration with steering is available within budget.

---

## 8.9 Product and pricing

**US-45 · Pick the right product · Critical**
As an Etsy seller, I want product options ranked by demand, competition and profit, so that I choose on evidence rather than habit.
*Accepts:* Ranked configurations with three sub-scores · real production costs · resulting margin · recommended colours and sizes.

**US-46 · See real profit · Critical**
As an Etsy seller, I want an itemised profit calculation including every fee, so that I know what I actually earn.
*Accepts:* Every cost line itemised · advertising fees modelled as charged by default · both with-ads and without-ads outcomes shown · updates live with price.

**US-47 · Not price below viability · Critical**
As an Etsy seller, I want to be prevented from publishing below my minimum margin, so that I never sell at an unintended loss.
*Accepts:* Floor enforced · violations block rather than warn · the exact price needed to clear the floor is stated.

**US-48 · Price against the market · Important**
As an Etsy seller, I want to see my price against the competitive distribution, so that I position deliberately.
*Accepts:* Distribution shown with my price marked · warning if far outside the typical range.

**US-49 · Pick colours that work · Important**
As an Etsy seller, I want garment colours recommended from what sells, and validated against my artwork.
*Accepts:* Recommendations derived from the success cohort · warnings where the design will not read on the chosen colour.

---

## 8.10 SEO

**US-50 · Get listings that can be found · Critical**
As an Etsy seller, I want SEO built from what actually ranks in this niche, so that my listings get seen.
*Accepts:* Ten complete variations · keywords selected by measured competitor performance weighted by sales · every keyword shows its evidence.

**US-51 · Have real choices · Important**
As an Etsy seller, I want the ten variations to take genuinely different positions, so that I can choose a strategy rather than a wording.
*Accepts:* Each labelled with its positioning axis · variations are demonstrably distinct · each is scored and ranked.

**US-52 · Never be rejected · Critical**
As an Etsy seller, I want the system to guarantee my listing meets marketplace rules, so that publishing never fails on a formatting error.
*Accepts:* All constraints validated before submission · live counters while editing · invalid content cannot be submitted.

**US-53 · Edit freely · Important**
As an Etsy seller, I want to edit any generated text, so that I keep my own voice.
*Accepts:* All fields editable · live validation · edits preserved through regeneration of other variations.

---

## 8.11 Publishing

**US-54 · Review everything before going live · Critical**
As an Etsy seller, I want one screen showing everything about a product before I publish, so that I can approve with confidence.
*Accepts:* Artwork, mockups, product, variants, pricing, profit, listing content and legal status all on one page · a checklist distinguishing blocking from advisory items · printable.

**US-55 · Never publish by accident · Critical**
As an Etsy seller, I want the system never to publish without my explicit approval, so that I retain full control of my shop.
*Accepts:* Listings are always created as drafts first · activation requires explicit confirmation naming the shop and the listing · no scheduled or automatic publishing exists.

**US-56 · Never get duplicates · Critical**
As an Etsy seller, I want absolute certainty that no duplicate listing will be created, so that a technical failure cannot damage my shop.
*Accepts:* Duplicates are prevented under retry, timeout and crash conditions · verified by deliberate failure testing.

**US-57 · Recover from partial failure · Critical**
As an Etsy seller, I want a partly-failed publish to be repairable, so that a hiccup does not cost me the work.
*Accepts:* Each publishing step reports its own status · targeted retry repairs only what failed · nothing already created is duplicated · no regeneration required.

**US-58 · Not lose work when a connection drops · Important**
As an Etsy seller, I want everything preserved if my shop authorisation expires, so that reconnecting resumes rather than restarts.
*Accepts:* Authorisation loss pauses work · a clear reconnection prompt appears · work resumes after reconnection.

**US-59 · Save for later · Important**
As an Etsy seller, I want to save a finished product for review later, so that I can batch my approvals.
*Accepts:* Drafts persist indefinitely · a review queue exists · nothing expires unexpectedly.

---

## 8.12 Analytics and learning

**US-60 · See what sold · Critical**
As an Etsy seller, I want performance tracked automatically for everything I publish, so that I learn without maintaining a spreadsheet.
*Accepts:* Views, favourites, orders, revenue and status updated regularly · history retained · derived measures computed.

**US-61 · Compare prediction to reality · Critical**
As an Etsy seller, I want to see how accurate the system's predictions were, so that I know how much to trust them.
*Accepts:* Predicted scores plotted against actual outcomes · accuracy quantified · failures shown honestly, not hidden.

**US-62 · Know which strategy works for me · Important**
As an Etsy seller, I want to know whether success-derived or gap-derived designs perform better for me, so that I focus my effort.
*Accepts:* Direct comparison by concept origin · statistically caveated where samples are small.

**US-63 · Trace any product to its origin · Important**
As an Etsy seller, I want to trace any listing back through its artwork, concept, originating insight and research run, so that I can understand and repeat my successes.
*Accepts:* Full navigable chain · every step clickable · no broken links.

**US-64 · Control my spending · Critical**
As an Etsy seller, I want to see exactly what I have spent and on what, so that I can manage costs deliberately.
*Accepts:* Spend by activity, provider and period · against budget · most expensive activities highlighted · month-end projection.

**US-65 · Have the system improve · Important**
As an Etsy seller, I want the system to learn from my actual results, so that it becomes more accurate the longer I use it.
*Accepts:* Improvements derived from real outcomes · tested against held-back history · presented as a proposal with evidence.

**US-66 · Stay in control of the learning · Critical**
As an Etsy seller, I want to approve any change to how the system scores, so that it never changes behaviour behind my back.
*Accepts:* Changes require explicit approval · exactly what would change is shown · previous configurations are retained and revertible · historical scores are never altered.

**US-67 · Be told when learning is not yet possible · Important**
As an Etsy seller, I want to be told plainly when there is not enough data to learn from, so that I am not given false sophistication.
*Accepts:* Below the minimum, fitting is refused and the reason stated · the number of outcomes needed is shown.

**US-68 · Know when something goes wrong · Important**
As an Etsy seller, I want to be notified about first sales, deactivated listings and listings with no views, so that I can act promptly.
*Accepts:* Notifications for first sale, deactivation, zero views after a defined period, budget thresholds and publishing failures.

---

## 8.13 Cross-cutting

**US-69 · Never be surprised by cost · Critical**
As an Etsy seller, I want to see what an action costs before I take it, so that I always spend deliberately.
*Accepts:* Cost shown before every spending action · updates live as selections change · running total visible.

**US-70 · Stop something in progress · Important**
As an Etsy seller, I want to cancel a running operation, so that I am not forced to pay for research I no longer want.
*Accepts:* Cancellation available at any point · spending stops promptly · completed work is preserved.

**US-71 · Watch progress meaningfully · Important**
As an Etsy seller, I want to see what the system is actually doing, so that a nine-minute wait does not feel like a failure.
*Accepts:* Named steps with status · specific activity messages · results appear as they complete · running cost visible.

**US-72 · Keep working when a source fails · Critical**
As an Etsy seller, I want useful results even when a data source is unavailable, so that an outage does not block my day.
*Accepts:* Runs complete with reduced scope rather than failing · the limitation is prominently labelled · the fix is stated · affected conclusions are individually marked.

**US-73 · Find anything · Nice-to-have**
As an Etsy seller, I want to search across all my research, concepts, artwork and listings, so that past work remains accessible.
*Accepts:* Single search across all entity types · results grouped and navigable.

**US-74 · Own my data · Important**
As an Etsy seller, I want to export everything, so that I am never locked in.
*Accepts:* Reports export as documents · data exports as spreadsheets · artwork downloads in full resolution · a complete archive is available.

---

# 9. MVP Definition

## 9.1 MVP principle

> **The MVP is a complete, genuinely useful tool for one person — not a demonstration of a future SaaS.**

The test is simple: **would the creator choose to use this every day instead of their current process?** If not, it is not finished, regardless of how many features it contains.

A second test governs cuts: **does removing this feature break the loop from evidence to outcome?** If yes, it stays. If no, it can wait.

## 9.2 In the MVP

### Research and analysis — the whole thing, no compromises
- Niche and product type input with style and depth options
- Sub-niche discovery and ranking
- Opportunity scoring with five explained sub-scores and a verdict
- Seasonality profiling
- Competitor shop discovery, selection and ranking, with the 20/10/5 fallback
- Full listing collection with all attributes
- Visual analysis: palette, typography, layout, mockup style
- Keyword and tag analysis
- Success analysis with baselines, ratios, sample sizes and confidence
- Failure analysis with causality labelling
- Market gap detection with the demand floor
- All four report types with full evidence access

**Nothing here is cut.** This is the product's foundation — everything else is downstream of it, and a partial version would produce partial conclusions, which is worse than none.

### Creation — complete, with all safety gates
- 20 concepts (10 success-derived, 10 gap-derived) with reasoning and citations
- Design Success scoring on five dimensions with contribution breakdown
- Concept selection, regeneration and manual entry
- **Full legal and safety screening — mandatory, no shortcuts**
- Artwork brief generation and editing
- Multi-variant artwork generation
- Background removal, upscaling, cropping
- Print-readiness validation with specific remedies
- Originality checking
- Renditions and downloads

**The legal gate is not optional in the MVP.** Shipping without it would expose the user to the single risk that can end their business, and retrofitting it into a system that already generates artwork is far harder than building it in.

### Commerce — complete for the six supported product types
- Product configuration ranking
- Full cost breakdown with advertising fees modelled as charged
- Price solving with margin floor enforcement
- Colour recommendation and artwork compatibility validation
- Ten differentiated SEO variations with evidence and validation
- Printify product creation, placement and mockups
- Etsy draft creation, images, variants and pricing
- Final review with a hard/soft checklist
- Approved publishing with duplicate prevention

### Feedback — enough to close the loop
- Daily performance tracking
- Portfolio and per-niche dashboards
- Prediction-versus-outcome comparison
- Cost tracking against budget
- Full traceability from listing back to research
- Success-derived versus gap-derived comparison

### Platform essentials
- Single-user authentication with a second factor
- Encrypted credential storage
- Integration health monitoring
- Spending limits enforced before spending
- Live progress with cancel and resume
- Graceful degradation with honest labelling
- Notifications
- Data export

## 9.3 Deliberately excluded from the MVP

| Excluded | Reason |
|---|---|
| **Multiple users, teams, roles** | One user. Building team features now serves nobody and delays what does. |
| **Subscriptions and payments** | Nothing to sell until the product is proven. |
| **Automatic learning weight adjustment** | Meaningful learning needs 50+ published outcomes, which is 3–6 months of use. The infrastructure to *record* outcomes ships in the MVP; the fitting can follow. |
| **Additional marketplaces** | Etsy's specific mechanics are baked into the analysis. A second marketplace is a second product's worth of work. |
| **Advertising management** | Different problem, different data, different skill. |
| **Bulk publishing** | Encourages the volume strategy this product exists to replace. |
| **Automatic publishing without approval** | Violates a core principle. Not "later" — never. |
| **Mobile applications** | The product is a data-dense analysis tool. It is a desktop product. |
| **Non-English listings** | Doubles the SEO problem for no current user. |
| **Video and animated assets** | Not required for the core loop. |
| **Design template library** | The product generates originals; a template library contradicts its premise. |
| **Public API** | No consumer exists. |
| **White-label reporting** | Agency feature, no agency users. |

## 9.4 MVP acceptance criteria

The MVP is complete when **all** of the following hold:

1. A research run on a real niche produces all four reports in under 11 minutes for under £1.20.
2. Every finding displays cohort prevalence, market baseline, ratio, sample size and confidence.
3. A run continues and produces clearly-labelled partial results when any single data source is disabled.
4. Twenty distinct concepts are generated, each citing evidence that resolves correctly.
5. An unscreened or blocked concept cannot produce artwork — verified by attempting it at the system level, not just through the interface.
6. A selected concept produces four print-ready variants for under £0.60 including retries.
7. Artwork failing print validation cannot be attached to a product.
8. A product publishes end-to-end to a live Etsy listing.
9. Deliberately induced failures during publishing produce zero duplicates.
10. Publishing is impossible while any hard checklist item fails, enforced by the system.
11. Performance data flows back and any listing traces fully to its originating research.
12. Total time from fresh niche to published product is under 30 minutes elapsed and under 15 minutes of attention.
13. **The creator prefers using it to their previous process.**

Criterion 13 is not decorative. It is the actual bar.

## 9.5 What would make the MVP a failure

- Reports that confirm what the user already knew rather than changing decisions.
- Artwork requiring more rework than designing from scratch.
- Products published through the system performing no better than the manual baseline after 90 days.
- The user reverting to their old process for any stage.

Each of these is worth discovering early, and each is a legitimate reason to stop or pivot rather than continue building.

---

# 10. Future Expansion Plan

Expansion is sequenced by **evidence**, not by ambition. Each horizon has an entry condition.

## Horizon 1 — Depth for the single user
*Entry condition: MVP in daily use for one month.*

| Feature | Rationale |
|---|---|
| **Active learning loop** | The recording infrastructure ships in the MVP; once 50+ outcomes exist, fitting becomes meaningful. This is the highest-value follow-on because it is the product's compounding mechanism. |
| **SEO A/B testing** | Nine unselected variations already exist per product. Testing them against live listings converts stored data into measured knowledge. |
| **Run comparison over time** | Re-research a niche and see what changed: which competitors entered, which factors shifted, whether the opportunity improved. |
| **Portfolio-wide margin monitoring** | Production costs change. A silent margin erosion across 200 listings is a real and invisible risk. |
| **Order-level cost reconciliation** | Stop *modelling* margin and start *measuring* it against actual charges. |
| **Refined proxy models** | Use the user's own realised sales to calibrate estimates, materially improving the public-data-only path. |
| **Bulk operations with per-item review** | Batch approval without abandoning per-item checks. |

## Horizon 2 — Multi-user foundations
*Entry condition: 30+ products published through the system, and the value articulable in one sentence a stranger would pay for.*

| Feature | Rationale |
|---|---|
| **Multiple users and roles** | The approval gate built for safety becomes the delegation primitive teams need. |
| **Subscription billing with usage components** | Every run costs real money. Flat-rate unlimited pricing is how AI products go bankrupt. |
| **Separate workspaces** | Agencies need client isolation; scaling sellers need shop separation. |
| **Shareable reports** | The reports are the most naturally shareable artefact the product produces. |
| **Onboarding for non-technical users** | The MVP assumes a user who can generate an API token. Real customers cannot. |
| **Guided data-source setup** | The market data question is genuinely subtle and needs an opinionated, forgiving flow. |

## Horizon 3 — Platform intelligence
*Entry condition: paying customers with recurring usage.*

| Feature | Rationale |
|---|---|
| **Cross-account benchmarking** | *"Your opportunity score of 68 is in the top 30% of gardening niches researched on this platform."* Only a multi-user product can offer this, and it is a genuine differentiator — with a carefully drawn privacy boundary. |
| **Shared market intelligence** | Two users researching the same niche in the same week should not both pay to analyse the same competitor images. This roughly doubles gross margin at scale and is the single largest unit-economics improvement available. |
| **Trend alerts** | Notify when a tracked niche's opportunity score changes materially. |
| **Automated re-research** | Keep niche intelligence current without manual re-runs. |
| **Advanced learning** | More sophisticated models once sufficient cross-account outcome data exists. |
| **Public API** | Enables agency workflows and integrations. |

## Horizon 4 — Category expansion
*Entry condition: strong retention and profitable unit economics on the core product.*

| Feature | Rationale |
|---|---|
| **Additional marketplaces** | The intelligence engine is marketplace-agnostic in principle. Amazon Merch and Shopify are the obvious next steps. |
| **Additional product categories** | Phone cases, stickers, blankets, jewellery — each requires catalogue and placement work, not new intelligence. |
| **Shop-level strategy** | Move from "which product?" to "what should my whole shop be?" — portfolio analysis, gap-filling across an existing catalogue. |
| **Competitive monitoring** | Track specific competitors over time and alert on their moves. |
| **Advertising intelligence** | Attribute ad spend to product characteristics — a natural extension once outcome data is rich. |

## Never on the roadmap

Recorded so nobody proposes them twice:

| Never | Why |
|---|---|
| Automatic publishing without human approval | Violates the core principle. The human owns the irreversible. |
| Volume listing tools | Directly contradicts the product's thesis. |
| Design template libraries | The product generates originals; templates would undermine that. |
| Any technique designed to evade detection when gathering data | Non-negotiable ethical line. If access requires evasion, the capability is unavailable. |
| Copying or tracing competitor artwork | The product's originality guarantee is structural and must remain so. |
| Selling or sharing users' outcome data | It is the users' asset, held in trust. |

---

# 11. Feature Prioritisation

Four tiers, with explicit reasoning for each placement.

## 🔴 Critical — the product does not exist without these

| Feature | Why critical |
|---|---|
| Niche and product input with depth control | The entry point. No input, no product. |
| Opportunity scoring with explained sub-scores | The first and most valuable answer the product gives. Delivers value even if everything downstream were removed. |
| Sub-niche discovery and ranking | Broad niches are unactionable. Specificity is what makes the research usable. |
| Competitor shop discovery and selection | Every downstream analysis depends on having the right shops. |
| Full listing data collection | The raw material for all analysis. |
| Visual analysis of competitor designs | **The core differentiator.** No other tool converts aesthetics into statistics. Without this the product is a worse EverBee. |
| Success analysis with baselines and confidence | The central promise: knowing what works. |
| **Baseline display on every finding** | Without the baseline, findings are actively misleading. This is not a nicety — it is what separates analysis from noise. |
| **Statistical suppression of weak findings** | A confident report built on twelve data points is worse than no report. |
| Failure analysis | Half the insight, and the half nobody else provides. |
| Market gap detection with demand floor | The highest-leverage strategic output. The floor is critical because without it the engine recommends deserts. |
| Concept generation grounded in evidence | The bridge between research and creation — the thing nothing else does. |
| Design Success scoring with reasoning | Turns twenty ideas into a prioritised shortlist. |
| **Legal and safety screening before generation** | The only feature protecting against an existential risk. Non-negotiable. |
| Artwork generation with print validation | Without validated print readiness, the output is not usable and the automation promise fails. |
| Originality checking | The product's originality claim must be verifiable, not asserted. |
| Product recommendation with real costs | Wrong product choice destroys margin regardless of design quality. |
| Profit calculation with advertising fees modelled | Optimistic margin arithmetic is how POD sellers lose money. |
| Margin floor enforcement | Prevents unintentional loss-making. |
| SEO generation with evidence-based keywords | A listing nobody finds does not exist. |
| Marketplace constraint validation | Publishing must never fail on a formatting error. |
| Printify and Etsy publishing | Without automation the 45-minute manual process remains and the value proposition collapses. |
| Draft-first with human approval | Core principle. Also the primitive that later enables team workflows. |
| **Duplicate prevention** | The most operationally damaging failure the system could cause. |
| Final review screen with hard checklist | The last line of defence before an irreversible action. |
| Performance tracking | Without outcomes there is no loop, and without the loop there is no moat. |
| Full traceability | Enables both learning and trust. |
| Spending limits enforced before spending | The cost of AI operations makes this a safety feature, not a convenience. |
| Cost visibility before action | Users must never be surprised by a charge. |
| Graceful degradation with honest labelling | External dependencies will fail. The product must survive it and say so. |
| Cancel and resume | Runs cost money and take minutes. Both must be interruptible and recoverable. |

## 🟠 Important — significantly degraded without these, but shippable

| Feature | Why important, not critical |
|---|---|
| "Median winning listing" synthesis | Enormously useful, but derivable from the factor list by a determined user. |
| Do/Avoid printable sheet | Presentation of existing data. |
| Causality labelling on failure findings | Improves interpretation quality; the findings remain valuable without it. |
| Interaction effects | Genuine additional insight, but second-order. |
| Automatic style selection | Users can choose manually; automation is better but not essential. |
| Concept regeneration with steering | Improves output quality; the initial twenty are usable. |
| Artwork brief editing | Gives creative control; defaults are workable. |
| Multiple artwork variants | Improves hit rate; one good variant suffices. |
| Vectorisation | Valuable for typography work; raster output is publishable. |
| Competitor price context | Aids positioning; the margin calculation is what actually protects the user. |
| Free-shipping strategy | A meaningful optimisation, not a blocker. |
| SEO variation differentiation axes | Improves choice quality; ten variations have value regardless. |
| Listing preview | Reduces errors; validation already prevents rejection. |
| Run comparison over time | Powerful once history exists; irrelevant on day one. |
| Prediction accuracy view | Builds trust; the predictions function without it. |
| Origin comparison (success vs gap) | Directly actionable, but requires accumulated data. |
| Notifications | Improves responsiveness; the dashboard covers it. |
| Data export | Important for trust and portability; not needed to operate. |
| Global search | Becomes important as data accumulates. |

## 🟡 Nice-to-have — real value, deferrable without harm

| Feature | Reasoning |
|---|---|
| Manual concept entry | Serves a narrow but real use case. |
| Concept expansion into variants | Useful for exploration; the base twenty suffice. |
| "Crowded loser" analysis | Interesting and occasionally valuable; a refinement of failure analysis. |
| Visual galleries of winning palettes and typography | Pleasant and helpful; the data is already in the factor list. |
| Coverage heat map | An alternative view of gap data. |
| Best-time-to-publish recommendation | Derivable from the seasonality chart. |
| Percentile comparison against prior niches | Nice context, requires history. |
| Review queue for drafts | Workflow convenience for higher volumes. |
| Batch publishing | Deliberately deprioritised — it encourages the volume strategy this product opposes. |
| Recently-collected data reuse | A cost optimisation, invisible when working. |
| Image count recommendation | A single useful number, derivable from the analysis. |

## 🔵 Future — explicitly out of scope now

| Feature | When |
|---|---|
| Active learning weight recalibration | Horizon 1 — needs 50+ outcomes |
| SEO A/B testing on live listings | Horizon 1 |
| Order-level cost reconciliation | Horizon 1 |
| Portfolio margin monitoring | Horizon 1 |
| Multiple users, teams, roles | Horizon 2 |
| Subscription billing | Horizon 2 |
| Separate client workspaces | Horizon 2 |
| Shareable and white-labelled reports | Horizon 2 |
| Cross-account benchmarking | Horizon 3 |
| Shared market intelligence caching | Horizon 3 |
| Trend alerts and automatic re-research | Horizon 3 |
| Public API | Horizon 3 |
| Additional marketplaces | Horizon 4 |
| Additional product categories | Horizon 4 |
| Shop-level portfolio strategy | Horizon 4 |
| Competitive monitoring | Horizon 4 |
| Advertising intelligence | Horizon 4 |

## 11.5 Prioritisation principles applied

1. **Anything protecting against catastrophic risk is Critical** regardless of frequency of use. Legal screening runs on every concept but matters on one in fifty — and that one can end a business.
2. **Anything preventing the user from being misled is Critical.** Baseline display and statistical suppression are unglamorous and absolutely essential, because the failure mode is a confident wrong answer.
3. **Anything the user cannot do themselves is prioritised above anything they can.** Visual analysis of 400 listings is impossible manually; a prettier chart is not.
4. **Anything that closes the evidence-to-outcome loop is Critical**, because the loop is the moat.
5. **Presentation improvements are Important at best.** Data first, polish second.
6. **Features that contradict the product thesis are never built**, however often they are requested.

---

# 12. Product Differentiation

## 12.1 Against Etsy research tools (EverBee, eRank, Alura, Sale Samurai)

**What they do well.** Provide raw marketplace data — sales estimates, revenue estimates, keyword volume, competition metrics — usually as searchable tables and per-listing lookups.

**Where they stop.** They give you *numbers*. They do not tell you what to do with them.

| | Research tools | POD Intelligence |
|---|---|---|
| Raw market data | ✅ | ✅ (uses theirs where available) |
| Single go/no-go verdict | ❌ | ✅ |
| **Visual/aesthetic analysis** | ❌ | ✅ |
| **Failure analysis** | ❌ | ✅ |
| **Statistical rigour with baselines** | ❌ | ✅ |
| Gap detection | Partial | ✅ with demand floor |
| Design generation | ❌ | ✅ |
| Legal screening | ❌ | ✅ |
| Listing generation | Keyword suggestions | ✅ complete listings |
| Publishing | ❌ | ✅ |
| **Outcome learning** | ❌ | ✅ |

**The decisive difference.** A research tool can tell you that a shop makes £8,000 a month. It cannot tell you that shops making £8,000 a month in this niche use muted green palettes at 2.1× the market rate, price at £22.95, use nine images, and never use script typography. That requires analysing every listing across multiple dimensions and computing lift against a baseline — which is a different kind of product, not a better version of the same one.

**And critically:** a research tool cannot tell you what *fails*, because failed listings are invisible to search.

## 12.2 Against AI image generators (Midjourney, Ideogram, DALL·E)

**What they do well.** Produce high-quality images from text prompts.

**Where they stop.** They have no idea what sells. They optimise for aesthetic quality against a global prior, identical for every user.

| | Image generators | POD Intelligence |
|---|---|---|
| Image quality | ✅ | ✅ (uses one) |
| **Market context** | ❌ | ✅ |
| **Data-derived palettes** | ❌ | ✅ |
| **Style chosen by measured performance** | ❌ | ✅ |
| Print readiness validation | ❌ | ✅ |
| Transparent background | Sometimes | ✅ guaranteed and verified |
| 300 DPI at true print size | Manual | ✅ automatic |
| **Legal screening** | ❌ | ✅ |
| **Originality verification** | ❌ | ✅ against the actual market |
| Concept development before generation | ❌ | ✅ |
| Prediction of commercial success | ❌ | ✅ |

**The decisive difference.** An image generator answers *"what would look good?"* This product answers *"what would sell here?"* — and those are different questions with different answers.

There is also a compounding problem the generators create: as everyone uses the same tools with the same default prompts, output converges. Designs derived from niche-specific measured evidence diverge from that convergence by construction.

**And the practical difference:** a beautiful image with a white background at 1024×1024 is not a product. Making it one takes 40 minutes of manual work per design.

## 12.3 Against SEO tools (Marmalead, eRank keyword tools)

**What they do well.** Keyword volume, competition scores, tag suggestions.

**Where they stop.** They optimise text in isolation from the product it describes.

| | SEO tools | POD Intelligence |
|---|---|---|
| Keyword data | ✅ | ✅ |
| **Keywords weighted by the sales of listings using them** | ❌ | ✅ |
| Complete listings, not just keywords | ❌ | ✅ |
| **Multiple differentiated positioning strategies** | ❌ | ✅ ten labelled axes |
| Evidence shown per keyword | ❌ | ✅ |
| Connected to the design being sold | ❌ | ✅ |
| Constraint validation before publishing | Partial | ✅ |
| Legal screening of listing text | ❌ | ✅ |
| Direct publishing | ❌ | ✅ |

**The decisive difference.** A keyword tool tells you "gardening gifts" has volume 12,000 and competition 8,400. It cannot tell you that among listings actually *selling* in this niche, "allotment gift" appears in 34% of top performers against a 9% baseline. Volume measures searches; this product measures *what the winners actually use*.

## 12.4 Against manual research

**What manual research does well.** Nuance, intuition, and pattern recognition a human genuinely has and software genuinely does not.

**Where it fails.** Not on insight — on **systematic processing**.

| Task | Manual | POD Intelligence |
|---|---|---|
| Evaluate a niche | 2–4 hours | 4 minutes |
| Analyse 20 shops' full catalogues | ~10 hours | 5 minutes |
| Compute base rates across 400 listings | Practically impossible | Automatic |
| Find failed listings | Structurally invisible | Systematic |
| Check trademarks per concept | 20 minutes, usually skipped | Automatic |
| Validate print readiness | Ad hoc | Automatic |
| Cross-reference tags against sales | ~4 hours | Automatic |
| Publish one product | 45 minutes | 2 minutes |
| Track outcomes longitudinally | Nobody does this | Automatic |

**The decisive difference.** Not that the software is smarter. It is that these are all tasks where the bottleneck is **volume of systematic processing**, and humans are unable to do them at the required scale no matter how skilled or diligent.

A human cannot count 400 listings by palette. Not "would rather not" — cannot, reliably, repeatedly, for every niche they consider.

## 12.5 The combination advantage

The four capabilities individually are all available elsewhere. **The combination is not available anywhere**, and the combination is where the value is:

```
        DATA ANALYSIS
   (what the market actually is)
              │
              ▼
       AI REASONING
   (what that data means)
              │
              ▼
     DESIGN GENERATION
   (turning meaning into products)
              │
              ▼
        AUTOMATION
   (getting products to market)
              │
              ▼
       ┌─── OUTCOMES ───┐
       │                │
       └── feeds back ──┘
```

**Why the chain matters more than the links.**

1. **Each stage improves the next.** Better analysis produces better concepts. Better concepts produce better artwork. Better artwork with better SEO produces better outcomes. Better outcomes improve the analysis. Break any link and the compounding stops.

2. **The handoffs are where value is lost manually.** A seller with excellent research and excellent design skills still loses most of the value in the transfer — the insight does not survive the walk from spreadsheet to canvas. Automating the handoff is worth more than improving either end.

3. **Only an integrated system can close the loop.** Feeding outcomes back requires knowing which design attributes produced which result. That linkage only exists if one system owns both ends.

4. **The moat is at the end of the chain.** Anyone can build stage one. The dataset linking *design attributes → market conditions → realised sales* can only be built by owning the whole chain, and it compounds with every use.

## 12.6 The honest limitations

Stated because credibility requires it, and because the team should design with these in view.

| Limitation | Reality |
|---|---|
| **Sales figures are estimates** | Third-party estimates carry real error. Mitigated by using rank-based rather than value-based analysis — cohorts and lift are largely robust to systematic bias that affects all listings similarly — and by labelling estimates everywhere. |
| **Correlation is not causation** | Muted greens correlating with success does not prove greens cause sales. The product labels causality honestly and never overclaims. |
| **The market moves** | Today's evidence describes today. Analysis has a shelf life, which is why re-research and comparison exist. |
| **AI artwork is not a replacement for a great designer** | It is a replacement for *no designer*, and for the generic templates most sellers currently use. |
| **Legal screening reduces risk, it does not eliminate it** | Stated prominently and permanently. It is not legal advice. |
| **The learning loop needs time** | Meaningful recalibration requires 50+ outcomes, realistically 3–6 months. The product says so plainly rather than pretending to learn from twelve data points. |

**A product that overclaims will be caught out by its own analytics dashboard.** The prediction-accuracy view exists precisely to keep the product honest with its user, and it would be self-defeating to market claims the dashboard will later contradict.

---

# 13. Success Metrics

## 13.1 The metrics that actually matter

Ranked by how well they indicate real value.

### Tier 1 — Does it make the user money?

| Metric | Baseline | 6-month target | Measured by |
|---|---|---|---|
| **Revenue per published listing** | user's current average | **+100%** | Comparing listings created through the system against the pre-existing portfolio |
| **Share of listings making a sale within 90 days** | ~20% typical | **≥ 45%** | Direct measurement |
| **Revenue per hour of work** | user's current | **+300%** | Revenue attributed, divided by hours spent |
| **Products published per week** | 5–10 | **15–25** | Direct count |

**Revenue per published listing is the single most important number in this document.** If it does not roughly double, the product has failed at its actual purpose, regardless of how elegant everything else is.

### Tier 2 — Does it save time?

| Metric | Baseline | Target |
|---|---|---|
| Time to evaluate a niche | 2–4 hours | **< 5 minutes** |
| Time to analyse competitors properly | ~10 hours | **< 10 minutes** |
| Time from concept to print-ready artwork | 1–3 hours | **< 5 minutes** |
| Time to write and validate a listing | 20–30 minutes | **< 2 minutes** |
| Time to publish one product | 45 minutes | **< 2 minutes** |
| **Total active attention per published product** | ~2 hours | **< 15 minutes** |

### Tier 3 — Is it accurate?

| Metric | Target | Meaning |
|---|---|---|
| **Prediction accuracy** | Meaningful positive correlation between Opportunity Score and realised outcome after 100 published products | The scores are real, not decorative |
| **Precision at the top** | Of the ten designs the system scored highest, at least five in the top quartile of outcomes | Practical usefulness of the ranking |
| **Calibration improvement** | Measurably better after the first learning cycle | The loop works |
| **Factor validation** | At least 60% of high-confidence success factors show predictive value in realised outcomes | The statistics reflect reality |

**A prediction with no measured accuracy is decoration.** This tier is what separates a real analytical product from a plausible-sounding one, and the product must report these numbers to the user whether or not they are flattering.

### Tier 4 — Is the output good enough?

| Metric | Target | Meaning |
|---|---|---|
| Artwork first-pass acceptance | **> 55%** | Artwork is usable without rework |
| Print validation pass rate | **> 80%** first attempt | The pipeline actually produces printable files |
| Concept selection rate | **3–6 of 20** | The concepts are good enough to choose from, but not indiscriminately |
| SEO edit rate | **< 40%** | Heavy editing means the SEO engine is weak |
| Publish success without manual fix | **> 95%** | The automation genuinely automates |
| Listings removed for IP issues | **Zero** | The legal gate works |

The SEO edit rate is a particularly good signal. If the user rewrites most listings, the generation is not good enough, however sophisticated the keyword analysis.

### Tier 5 — Does it perform and cost what it should?

| Metric | Target |
|---|---|
| Research run (Standard) | < 11 minutes, < £1.20 |
| Opportunity report available | < 4 minutes |
| Artwork per accepted design | < £0.60 |
| Marginal cost per published product | < £1.80 |
| Run completion without intervention | > 97% |
| **Duplicate listings created** | **Zero** |
| Monthly infrastructure | < £140 |

### Tier 6 — Does the user actually want it?

| Metric | Target |
|---|---|
| **Would the creator return to their old process?** | **No** — the binary test |
| Research runs per week | 3–8 sustained |
| Weeks used out of weeks available | > 80% |
| Stages bypassed in favour of manual work | Zero sustained |
| Reports exported or referenced repeatedly | Frequent — indicates real utility |

## 13.2 Leading indicators

Available before revenue data accumulates, and useful for steering during development.

| Indicator | Healthy | Concerning | What a concern means |
|---|---|---|---|
| Concepts selected per run | 3–6 of 20 | < 2 or > 12 | Too few: concepts are poor. Too many: the user is not discriminating, or the scores are not differentiating. |
| Artwork regeneration rate | < 1.5 per accepted | > 3 | Briefs are not specific enough |
| SEO variations edited | < 40% | > 70% | The SEO engine does not understand the niche |
| Niches rejected after research | 20–40% | < 10% | Scores are not discriminating — everything looks good |
| Legal flags per 100 concepts | 5–15 | > 30 or < 2 | Too many: over-sensitive and annoying. Too few: probably missing real risk. |
| Time between research and publishing | < 2 days | > 1 week | The creation stage is too hard |
| Manual overrides of recommendations | < 30% | > 60% | The recommendations are not trusted |

**The niche rejection rate is especially informative.** If the system approves every niche, its scores are not doing any work. A product that sometimes says "do not enter this market" is more valuable than one that always says yes.

## 13.3 Metrics deliberately not used

| Not measured | Why |
|---|---|
| Number of listings published | Volume is the strategy this product exists to replace. Optimising for it would corrupt the product. |
| Concepts generated | Generation is cheap. Selection is the meaningful act. |
| Images generated | An input cost, not an outcome. |
| Time spent in the application | For a productivity tool, *less* is better. Engagement time is an anti-metric here. |
| Feature usage breadth | Using three features effectively beats using ten shallowly. |
| Total data collected | Volume without conclusions is not value. |

## 13.4 Review cadence

| Metric group | Review | Action if missed |
|---|---|---|
| Tier 1 outcomes | Monthly (from 90 days post-launch) | Fundamental reassessment |
| Tier 2 time savings | Per phase | Workflow redesign |
| Tier 3 accuracy | Per 50 outcomes | Scoring model review |
| Tier 4 quality | Weekly | Prompt and pipeline iteration |
| Tier 5 performance and cost | Continuous with alerting | Immediate investigation |
| Tier 6 satisfaction | Continuous self-observation | Honest reconsideration |

## 13.5 The final test

> **After 90 days of use, is the creator publishing fewer products and earning more money — and can they explain, for any listing in their shop, exactly why it exists?**

If yes, the product works.

Everything in this document exists to make that sentence true.

---

## Document control

| | |
|---|---|
| **Part** | 1 of the POD Intelligence architecture series |
| **Covers** | Product definition, users, journeys, requirements, prioritisation, metrics |
| **Excludes** | Technical architecture, data design, API design, technology selection, implementation |
| **Next** | Part 2 — System Architecture & Data Design |
| **Status** | Ready for engineering review |

### Open questions for the engineering team

These require answers before Part 2 can be completed:

1. **Market data access.** What is the realistic, compliant path to sales and revenue estimates? The product must work without them, but is materially better with them. This is the highest-impact open question.
2. **Statistical thresholds.** What minimum sample sizes and significance levels should govern the suppression of findings? Getting this wrong in either direction damages the product — too strict and it says nothing, too loose and it misleads.
3. **Legal screening scope.** Which registries and which goods classes constitute adequate coverage, and what risk appetite should the default settings encode?
4. **Print requirements per product type.** Exact dimensions, resolutions and placement rules for each of the six supported products.
5. **Fee model accuracy.** Confirmation of every marketplace fee component and its treatment, since every profit figure depends on it.
6. **Performance data availability.** What outcome data is reliably retrievable for the user's own listings, and at what frequency?
