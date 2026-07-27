# ADR-0001 — Multi-tenancy from day one

**Status:** accepted · **Date:** Phase 0 · **Deciders:** CTO

## Context

The product is specified as single-user for personal use, with an explicit requirement that it can later become a multi-user SaaS with subscriptions. The question is *when* to introduce the tenancy boundary.

Single-user systems that later add tenancy face a specific, well-documented failure: a `tenant_id` backfill across every table, rewriting every query, re-indexing every hot path, and auditing every code path for cross-tenant leakage — while live data exists. It is a multi-month project with genuine risk of data corruption and of silent leakage that is only discovered by a customer.

## Decision

Every domain table carries `workspace_id uuid NOT NULL` from the first migration. Every composite index leads with `workspace_id`. Every repository method takes a workspace-scoped context. Row-Level Security policies are written and CI-tested from day one, enforced by flipping the database role at the SaaS phase. `workspace_members` exists with a role enum and holds exactly one `owner` row.

Single-user mode is simply "one workspace with one member".

## Options considered

| Option | Verdict |
|---|---|
| Add tenancy later, when needed | Rejected — the retrofit is the single largest technical risk in the SaaS transition, and it arrives exactly when the business can least afford downtime |
| Full multi-tenancy now (invitations, roles enforcement, billing) | Rejected — builds unproven product surface; violates the Phase 6 gate |
| **Tenancy in the data model now, tenancy features later** | **Accepted** |
| Database-per-tenant | Rejected — operationally heavy at any scale we will reach before Stage 4; revisitable for enterprise isolation |

## Consequences

**Positive**
- SaaS conversion becomes additive (doc 21 §2): enable RLS, add invitations, add billing. Days, not months.
- `workspace_id` is the natural shard key, and no cross-workspace query exists in the application — making Stage 4 sharding tractable.
- Cross-tenant leakage is prevented by construction and testable from the first sprint.
- Per-workspace credentials, budgets and rate limits are natural rather than bolted on.

**Negative / accepted costs**
- One extra column on ~40 tables and one extra parameter on every repository method.
- Slightly larger indexes.
- Discipline required: a query that forgets the scope is a bug, and CI must catch it.

**Mitigation for the negative:** the repository layer is the only place Prisma is called (ADR-0014), so the scope is applied in one place per aggregate rather than at every call site.
