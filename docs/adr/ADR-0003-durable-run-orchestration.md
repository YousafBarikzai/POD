# ADR-0003 — Durable, resumable run orchestration

**Status:** accepted · **Date:** Phase 0

## Context

A research run takes 3–22 minutes, spans ~14 steps, calls five external providers, and **spends real money** — roughly £1.10 in AI and data per standard run, more with artwork. Steps have dependencies, some run in parallel, some are optional, and one step (concept selection) waits indefinitely for a human.

If the process dies at step 11, the operator must not lose steps 1–10 or pay for them twice.

## Decision

Every run is a **persisted state machine**. `runs` holds the aggregate status; `run_steps` holds one row per step with status, attempt count, dependencies, input hash, output pointer, cost and timing. The step runner:

1. Claims a step with `SELECT … FOR UPDATE SKIP LOCKED`.
2. Asserts dependencies succeeded.
3. Reserves budget before executing.
4. Executes the handler.
5. Persists output **and** status **and** cost in one transaction.
6. Enqueues newly-unblocked steps.

Steps heartbeat every 15 s; a reaper requeues steps whose worker died. The DAG is declarative data, not control flow. Human gates are a run status (`awaiting_selection`), not a blocked job.

## Options considered

| Option | Verdict |
|---|---|
| In-memory job chaining (async/await through the steps) | Rejected — a crash loses everything and re-spends. Unacceptable given real per-run cost. |
| BullMQ flows / job chaining | Rejected — dependency semantics live in Redis, which is ephemeral by design. Redis loss would lose run state. |
| **Custom durable state machine on Postgres + BullMQ** | **Accepted** |
| Temporal | Deferred — the right answer at Stage 4. Today it is a second control plane, a new deployment, and a new failure mode for a single-user system. The DAG definition ports directly when the time comes. |
| AWS Step Functions | Rejected — vendor lock-in against NFR-V1, and poor fit for long human gates. |

## Consequences

**Positive**
- A worker can be killed at any point; the run resumes without repeating successful work or re-spending.
- Every step's cost, duration, attempt count and failure reason are queryable — the observability and cost-attribution stories come free.
- `run_steps` doubles as the outbox: any step `pending` with satisfied dependencies is re-enqueueable, so a crash between DB commit and queue enqueue self-heals.
- Human gates require no special machinery.
- Adding a step is a data change plus a handler.

**Negative / accepted costs**
- ~600 lines of orchestration code we own and must test, including the reaper and the resume path.
- One database write per state transition — negligible at our volumes.
- Chaos testing is mandatory, not optional, because the guarantees are ours to prove.

**Verification:** a weekly chaos test kills a worker mid-run and asserts identical final output with zero duplicate side effects.
