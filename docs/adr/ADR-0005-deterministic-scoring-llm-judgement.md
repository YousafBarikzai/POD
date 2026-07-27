# ADR-0005 — Deterministic maths, LLM judgement

**Status:** accepted · **Date:** Phase 0

## Context

The product's central promise is *data-driven decisions rather than guessing*. It produces roughly a dozen distinct scores — Opportunity, five sub-scores, Gap, Design Success, five predictor dimensions, product and SEO quality — and the operator makes real financial decisions from them.

There is an obvious shortcut: ask a capable model to read the data and return a score with reasoning. It is fast to build, it produces plausible-sounding output, and it is fundamentally wrong for this product.

## Decision

**No number that appears in a score is produced by a language model.**

| Layer | Responsibility |
|---|---|
| `packages/domain` (pure TypeScript) | All arithmetic: normalisation, weighting, statistics, significance, lift, effect size, pricing, scoring, ranking |
| LLM | Classification into controlled vocabularies, entity extraction, creative generation, and prose explanation of a computed result |

Reasoning displayed in the UI is rendered from the **computed contribution vector**, not authored freely by a model (FR-708). It is therefore always exactly consistent with the score.

## Options considered

| Option | Verdict |
|---|---|
| LLM produces scores and reasoning | Rejected — see consequences below |
| LLM produces scores, code validates ranges | Rejected — validating the range of a fabricated number does not make it real |
| **Code computes, LLM classifies and explains** | **Accepted** |
| Trained ML model computes scores | Deferred — this is the Phase 5 destination for the *Conversion* dimension specifically, and it replaces coefficients within the same deterministic framework, not the framework itself |

## Consequences

**Positive**
- **Reproducibility.** Re-running scoring against stored features with the same config version yields byte-identical values (AC-200). This is a hard requirement and is unachievable with model-produced numbers.
- **Auditability.** Every score decomposes into named feature contributions. "Why is this 68?" always has a complete answer.
- **The learning loop is possible at all.** Fitting weights against outcomes requires the weights to exist as data (ADR-0006). A model that "just knows" cannot be recalibrated.
- **Testability.** Scoring is pure functions with property tests — monotonicity, bounds, invariance under irrelevant permutation. No network, no flakiness, milliseconds.
- **Cost.** Arithmetic is free. Roughly 30 scoring computations per run cost nothing instead of ~£0.30.
- **Honesty.** A model asked to score will always produce a confident number, including when the underlying sample is n=3. Code refuses, because the significance test refuses.

**Negative / accepted costs**
- More design work: every score needs an explicit formula, normalisation choice and weight (Appendix A). This is upfront intellectual effort the shortcut avoids.
- Less "magical" apparent sophistication in early demos.
- Formula changes require code changes rather than prompt edits — though this is arguably a benefit, since it forces review.

**The decisive argument:** a scoring system whose numbers cannot be reproduced, explained or improved is not a decision system. It is a random number generator with good manners.
