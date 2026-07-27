# ADR-0004 — Provider adapter boundary

**Status:** accepted · **Date:** Phase 0

## Context

The system depends on six external providers with very different stability characteristics:

| Provider | Stability |
|---|---|
| EverBee / market data | **No public API.** May change or become unavailable at any time |
| Etsy | Documented, versioned, but policy and API both evolve |
| Printify | Documented, moderately stable |
| Ideogram | Young product; pricing, quality and terms all volatile |
| AI provider | Models change frequently; capabilities and pricing shift |
| Trademark registries | Stable but heterogeneous across three jurisdictions |

Every one of these is a candidate for replacement over a three-year horizon. If vendor SDK calls and vendor data shapes leak into engines, each replacement becomes a cross-cutting refactor.

## Decision

Every external dependency sits behind an interface in `packages/adapters/<provider>`, with an identical internal shape (`index.ts`, `client.ts`, `mappers.ts`, `errors.ts`, `schemas.ts`, `fixtures/`) and a universal contract:

- Every response parsed by Zod at the boundary; internal code never sees an unvalidated shape.
- Retries, rate limiting, circuit breaking, timeouts, tracing and cost recording are adapter responsibilities, not engine responsibilities.
- Provider errors are mapped to the domain `ApiError` catalogue; raw provider text is logged, never surfaced.
- Credentials are fetched per call from `CredentialService`, never held in module scope.
- Fixture-backed tests; **CI never calls a live API.**

**Enforced by a dependency-cruiser rule and a Semgrep rule: no vendor SDK may be imported outside its own adapter package.** A violation fails the build.

## Options considered

| Option | Verdict |
|---|---|
| Call SDKs directly from engines | Rejected — fastest to write, most expensive to change; makes offline development and deterministic testing impossible |
| Thin wrapper functions | Rejected — no consistent contract means retry/rate-limit/cost handling is reimplemented (differently) six times |
| **Full adapter packages with a universal contract** | **Accepted** |
| Anti-corruption layer per bounded context | Rejected as over-engineering at this size; the adapter *is* the ACL |

## Consequences

**Positive**
- `MOCK_PROVIDERS=true` runs the entire pipeline offline, free, in seconds — the single biggest developer-velocity decision in the project.
- Replacing Ideogram is an adapter plus style-template mappings plus an eval run (doc 12 §13). Roughly 80% of the artwork pipeline is provider-agnostic.
- The market data provider *chain* (doc 11) is only possible because the boundary exists.
- Rate limiting, cost accounting and circuit breaking are uniform and provably applied.

**Negative / accepted costs**
- ~22 person-days of adapter work in the estimate (doc 19 §10) — a real, visible cost paid up front.
- One extra mapping layer between provider shapes and domain types.
- Fixtures must be maintained as provider responses evolve; a stale fixture can hide a real break. Mitigated by a monthly job that replays fixtures against live sandboxes in staging and diffs the schemas.
