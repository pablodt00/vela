# Vela — project context

This folder is the written context for Vela. It exists so that anyone — a person
reading the repo cold, or an AI assistant being given the project — can understand
what is being built, why the decisions were made the way they were, and what is
deliberately out of scope.

It is documentation of intent. Where it disagrees with the code, the code is what
runs, but the disagreement is a bug in one of the two and should be resolved rather
than ignored.

## Read in this order

| File | What it covers |
|---|---|
| [`01-overview.md`](01-overview.md) | What Vela is, who is building it, the constraints that shape everything else |
| [`02-architecture.md`](02-architecture.md) | Services, Kafka topics, invariants, data stores, runtime, memory budget |
| [`03-domain-model.md`](03-domain-model.md) | Payment lifecycle, ledger model, reconciliation, risk scoring, glossary |
| [`04-services.md`](04-services.md) | Each service in plain language, including the banking concepts behind it |
| [`05-observability.md`](05-observability.md) | Prometheus, Grafana, Loki, Alloy, Kafbat UI — and the configuration for each |
| [`06-conventions.md`](06-conventions.md) | Repo layout, event shape, error format, idempotency, naming, definition of done |
| [`07-roadmap.md`](07-roadmap.md) | Phases 0–7 with done-conditions, and the current status |
| [`08-decision-log.md`](08-decision-log.md) | Every structural decision, why it was made, what was rejected |
| [`09-backlog.md`](09-backlog.md) | The implementation plan: 20 epics and 131 task issues, in dependency order |

## Two things to know before reading anything else

**Vela does not move real money and never will.** There is no licence and no access
to real payment rails. The rails are simulated by `mock-bank`. Everything above the
rails — ledger correctness, reconciliation, risk scoring, exposure aggregation,
operational tooling — is built as if it mattered.

**It runs on one laptop.** That is a deliberate constraint, not a limitation to be
worked around. Memory is the binding constraint on every design decision in this
folder. See [`01-overview.md`](01-overview.md).

## Keeping this current

- The status line in [`07-roadmap.md`](07-roadmap.md) is updated when a phase completes.
- Structural decisions are appended to [`08-decision-log.md`](08-decision-log.md) as they
  are made, with the alternatives that were rejected. The rejected options are the
  useful part — they are what stops a decision being re-litigated six weeks later.
- `contracts/` is the source of truth for event shapes, not these documents. A schema
  change is a deliberate, versioned act.
