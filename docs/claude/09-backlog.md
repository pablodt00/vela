# Backlog

The implementation plan, as GitHub issues. Twenty epics, one hundred and thirty-one
tasks, ordered by the phases in [`07-roadmap.md`](07-roadmap.md).

Every task issue carries the same shape: **why this exists** (the concept or the
decision behind it), **what to do** as a checklist, **done when**, and usually
something to learn or a trap to avoid. They are sized at roughly half a day to two
evenings each.

## How to work the backlog

Work top to bottom. The order is the dependency order — `ledger-core` needs the
schema before the state machine, reconciliation needs `mock-bank` before it has
anything to reconcile.

Labels:

| Label | Meaning |
|---|---|
| `epic` / `task` | An epic is a container; tasks are the work |
| `phase:0` … `phase:7` | Which roadmap phase it belongs to |
| `P0` / `P1` / `P2` | Within a phase: must, should, could |
| `svc:*` | Which service |
| `lang:*` | Java, Go, Python, TypeScript or infra |
| `type:e2e` | End-to-end verification |

`P0` within a phase is what that phase's done-condition depends on. `P2` can slip to
the next phase without blocking anything.

## The epics, in order

| # | Epic | Phase | Tasks |
|---|---|---|---|
| [1](../../issues/2) | Repo skeleton and local runtime | 0 | 11 |
| [2](../../issues/3) | Contracts, schema registry and CI | 0 | 6 |
| [3](../../issues/4) | `fx-ingester` — market data in | 1 | 5 |
| [4](../../issues/5) | `ledger-core` — double entry, state machine, outbox | 1 | 12 |
| [5](../../issues/6) | Logs you can correlate (Loki + Alloy) | 1 | 3 |
| [6](../../issues/7) | The platform layer, locally (k3d + ArgoCD) | 2 | 5 |
| [7](../../issues/8) | `mock-bank` — the simulated rails | 3 | 9 |
| [8](../../issues/9) | `recon-service` — proving the books | 3 | 10 |
| [9](../../issues/10) | `risk-service` — decisions with reasons | 3 | 6 |
| [10](../../issues/11) | `gateway` — the front door | 4 | 5 |
| [11](../../issues/12) | `quote-service` — rates, spread, TTL, HMAC | 4 | 5 |
| [12](../../issues/13) | `webhook-dispatcher` — delivery that survives failure | 4 | 4 |
| [13](../../issues/14) | `exposure-aggregator` — Kafka Streams positions | 4 | 3 |
| [14](../../issues/15) | Distributed tracing across three languages | 4 | 3 |
| [15](../../issues/16) | `ops-console` — the internal tool | 5 | 5 |
| [16](../../issues/17) | `customer-app` | 5 | 3 |
| [17](../../issues/18) | The data layer — Parquet, dbt, DuckDB | 5.5 | 5 |
| [18](../../issues/19) | Metrics, dashboards, load and chaos | 6 | 3 |
| [19](../../issues/20) | The artifact — `make demo`, README, recording | 7 | 4 |
| [20](../../issues/21) | End-to-end scenarios and Claude verification | 7 | 10 |

## Decisions taken while writing the backlog

Two things the roadmap left open, now resolved. Both should be reflected in
[`08-decision-log.md`](08-decision-log.md) once acted on.

**Phase 1 stops at CONVERTED.** The roadmap flagged that with funding going through
`mock-bank`, the vertical slice cannot reach FUNDED without `recon-service` — and
offered two options. Taken: fake funding with a temporary
`POST /payments/{id}/fund` stub, which the roadmap calls "the lighter option".
The stub has an explicit deletion issue in Phase 3, so it cannot become permanent.

**`exposure-aggregator` lands in Phase 4.** The roadmap never assigned it a phase.
Phase 4 puts it after `ledger.entries` has real volume from Phase 3, and immediately
before the ops console consumes `exposure.snapshots` in Phase 5.

## Two things worth knowing about the shape of the backlog

**The invariants are the spine.** The six non-negotiable invariants in
[`02-architecture.md`](02-architecture.md) each get implemented in Phase 1 to 4 and
each get an end-to-end scenario in Epic 20. The last issue in the project is a
mapping table proving every one of them is covered, or documenting honestly why it
cannot be.

**Memory is tracked from the start, not at the end.** The roadmap warns twice that
the `demo` profile has to fit on the target laptop and that finding out in Phase 7 is
too late. So there is a measurement issue in Phase 0, meant to be re-run at the end
of every phase.
