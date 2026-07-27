# POD Intelligence

> An AI-powered personal automation platform for creating profitable Etsy Print-On-Demand products.

**Status:** Design phase. **No application code exists yet — this repository contains specification documents only.**

---

## Start here

Two completed specification documents, read in order.

### 📘 [Part 1 — Product Definition](docs/PART-1-product-definition.md)

**Read first.** A complete product requirements document covering:

1. Executive Summary
2. Product Vision
3. Target Users
4. User Problems
5. Complete User Journey (12 stages)
6. Functional Requirements
7. Non-Functional Requirements
8. User Stories (74)
9. MVP Definition
10. Future Expansion Plan
11. Feature Prioritisation
12. Product Differentiation
13. Success Metrics

It deliberately contains **no database design, no API design and no technology choices** — those belong to Phase 1B.

### 📗 [Phase 1B — Technical Architecture](docs/PHASE-1B-technical-architecture.md)

**Read second.** How the software will be built. Twelve parts:

| Part | Section | Contents |
|---|---|---|
| A | System Architecture | Frontend, backend, database, AI, integration, storage, auth, background processing and deployment tiers — and how every part communicates |
| B | Technology Decisions | Every choice with what was rejected and why, grounded in this product's specific demands |
| C | Database Design | ~35 tables with fields, keys, relationships and the reasoning behind each structural decision |
| D | API Design | Every endpoint group with purpose, inputs, outputs and security requirements |
| E | Backend Services | Thirteen services with responsibilities and enforced boundaries |
| F | AI Architecture | Capability tiers, prompt registry, output contracts, injection defence, cost engineering, evaluation gates |
| G | Integration Architecture | Adapter contract, the market-data provider chain, marketplace, fulfilment, registries |
| H | Background Job System | Queues, job and step states, error classification, retry policy, crash recovery |
| I | Security Architecture | Credential protection, authentication, authorisation, encryption, rate limiting, file security |
| J | Scalability Architecture | Four growth stages, what actually breaks first, multi-user path, isolation, subscription preparation |
| K | Project Structure | Monorepo layout with machine-enforced dependency boundaries |
| L | Deployment Architecture | Environments, pipeline, migrations, monitoring, logging, backups, runbooks |

It contains **no application code** — implementation begins in Phase 2.

---

## What the product does

A user enters a niche ("Gardening") and a product type ("T-Shirts"). The system then:

```
Researches the market        →  Is this niche worth entering? (0–100 score, five explained sub-scores)
Analyses competitors         →  Who is winning, and what are their listings actually made of?
Finds success patterns       →  "84% of top performers use muted greens — market baseline 40%, lift 2.1×"
Finds failure patterns       →  What reliably fails here, and why
Finds market gaps            →  Where is the unserved demand?
Generates concepts           →  20 original ideas grounded in that evidence, each scored and justified
Screens for legal risk       →  Mandatory, before any artwork is created
Generates artwork            →  Print-ready, transparent, 300 DPI, validated
Builds the product           →  Ranked configurations, real costs, enforced margin floor
Writes the listings          →  10 differentiated SEO variations from measured keyword performance
Publishes                    →  Printify + Etsy draft → human approval → live
Tracks and learns            →  Real outcomes feed back and improve future predictions
```

**The core idea:** every competitor tool owns one link in that chain. This product owns the chain — and the chain is where value compounds, because every published product feeds a growing dataset linking *design attributes → market conditions → realised sales*.

## Scope

**Initially:** single user, one Etsy shop, personal use. The MVP must be genuinely useful for one person, not a demonstration of a future SaaS.

**Architected to allow later:** multiple users, teams, subscriptions, and a product sold to other Etsy/POD creators. No product decision may obstruct that path, but no SaaS infrastructure is built before the single-user product is proven.

## Non-negotiable principles

1. **Evidence over opinion.** Every number shows the data behind it. Every prevalence figure shows the market baseline, so a big percentage can never mislead.
2. **Cheap before expensive.** Text concepts are always generated and human-approved before any image-generation spend.
3. **The human owns anything irreversible.** Publishing, spending and accepting legal risk are human decisions. The system never publishes autonomously.
4. **Original by construction.** Competitors are analysed statistically. Competitor imagery and text never enter a generative prompt.
5. **Degrade, never die.** A missing data source produces a smaller, clearly-labelled result — never an error.
6. **Honesty about uncertainty.** Estimates look different from measurements. Weak findings are suppressed, not dressed up.
7. **Speed is a feature.** Meaningful partial results within seconds, not a nine-minute spinner.

## Ethical and legal position

- Competitor data is used for **aggregate statistical analysis only** — never to reproduce, trace or derive artwork.
- A mandatory legal and safety gate runs **before** any artwork is generated and cannot be bypassed.
- Data acquisition uses compliant methods only. **No technique whose purpose is to avoid detection will be implemented** — if access requires evasion, the capability is simply unavailable.
- Only public commercial data is collected. No personal data of buyers, reviewers or shop owners.

---

## Supporting working drafts

The `docs/` directory also contains earlier and more granular technical drafts. **Phase 1B is the authoritative technical specification**; these supplement it with additional detail in specific areas (notably the scoring formulas in Appendix A and the prompt catalogue in Appendix B) and are retained as working material.

<details>
<summary>Show supporting drafts</summary>

| Document | Contents |
|---|---|
| [00 Executive Summary](docs/00-executive-summary.md) | Technical vision, key architectural decisions, cost model |
| [01 Product Requirements](docs/01-product-requirements.md) | Earlier PRD draft, superseded by Part 1 |
| [02 Functional Requirements](docs/02-functional-requirements.md) | Engineering-level FR list with acceptance criteria |
| [03 Non-Functional Requirements](docs/03-non-functional-requirements.md) | Measurable budgets and verification methods |
| [04 User Workflows](docs/04-user-workflows.md) | State machines and failure/resume paths |
| [05 UI Specification](docs/05-ui-specification.md) | Page-by-page screens and components |
| [06 Database Schema](docs/06-database-schema.md) | Full data design *(Part 2 material)* |
| [07 Entity Relationship Diagrams](docs/07-entity-relationship-diagrams.md) | ERDs by domain *(Part 2 material)* |
| [08 API Architecture](docs/08-api-architecture.md) | API design *(Part 3 material)* |
| [09 Service Architecture](docs/09-service-architecture.md) | Services, queues, orchestration |
| [10 AI Orchestration](docs/10-ai-orchestration.md) | Model tiering, prompts, evals, guardrails, cost control |
| [11 Market Data Integration](docs/11-everbee-integration.md) | Provider abstraction and compliance posture |
| [12 Artwork Generation](docs/12-ideogram-integration.md) | Briefs, generation, print-readiness pipeline |
| [13 Etsy Integration](docs/13-etsy-integration.md) | OAuth, drafts, publishing, rate limits |
| [14 Printify Integration](docs/14-printify-integration.md) | Catalogue, products, mockups, cost model |
| [15 Security Architecture](docs/15-security-architecture.md) | Threat model, credentials, prompt-injection defence |
| [16 Scalability Architecture](docs/16-scalability-architecture.md) | Scaling stages and real bottlenecks |
| [17 Deployment Architecture](docs/17-deployment-architecture.md) | Environments, CI/CD, backup, runbooks |
| [18 Folder Structure](docs/18-folder-structure.md) | Code organisation and boundaries |
| [19 Development Roadmap](docs/19-development-roadmap.md) | Phases, sprints, estimates |
| [20 Risks & Mitigations](docs/20-risks-and-mitigations.md) | Risk register |
| [21 SaaS Migration Strategy](docs/21-saas-migration-strategy.md) | Path to multi-user and subscriptions |
| [Appendix A — Scoring Models](docs/appendix/a-scoring-models.md) | Every score formula and calibration method |
| [Appendix B — Prompt Catalogue](docs/appendix/b-prompt-catalogue.md) | Every AI call and its contract |
| [Appendix C — Glossary](docs/appendix/c-glossary.md) | Domain terms and metric definitions |
| [Architecture Decision Records](docs/adr/) | Decisions and their rationale |

</details>

---

## Document series

| Phase | Title | Status |
|---|---|---|
| **1** | **Product Definition** | ✅ **Complete — ready for review** |
| **1B** | **Technical Architecture** | ✅ **Complete — ready for review** |
| 2 | Implementation — foundation and orchestration | Not started |
| 3 | Implementation — intelligence, creative, commerce, feedback | Not started |
