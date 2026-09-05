# Overview

## What Vela is

A portfolio-grade simulation of a cross-border payments and FX platform.

A customer requests a quote to convert one currency into another, accepts it, funds
it, and the platform converts, settles and reconciles the payment. Market data is
real where free sources allow. The banking rails are simulated.

Vela does not move real money and never will — no licence, no real rails. Everything
above the rails is built for real: ledger correctness, reconciliation, risk scoring,
exposure aggregation, operational tooling.

## Who is building it, and why that matters

A software engineer with roughly three years of experience, working in fintech in
Spain, building this in evenings and weekends. Day-to-day strengths are Python and
web development. Java, Go and Kafka are the areas being deliberately levelled up.

That shapes the project: the interesting parts are deliberately in the languages
being learned, and the scope of any given phase is set by what can be finished in
spare time rather than what would be architecturally maximal.

## The binding constraint: it runs locally

There is no cloud deployment and no budget for one. Everything runs on a laptop via
docker-compose, or on a local k3d/kind cluster. Kubernetes manifests are written and
kept valid from day one because they are part of the design, but they are exercised
locally.

Consequences that apply to every decision in this repository:

- **Memory is the binding constraint.** Never assume all services run at once. The
  system is designed so that a subset can be brought up: the service under edit plus
  its direct dependencies.
- **Every JVM service gets an explicit heap cap.** Never rely on JVM defaults.
- **One Redpanda broker. One Postgres container, with a database per service.** Not a
  broker per topic, not a Postgres instance per service.
- **No paid services, paid tiers, or anything requiring a credit card.** If a paid
  option is genuinely the only way to do something, that is stated plainly rather than
  worked around.
- If a pattern only makes sense with managed cloud infrastructure, the local
  equivalent is used instead and the difference is labelled.

## Language boundaries

These are not preferences. A service written in the wrong language here is a design
smell.

| Language | Domain | Why |
|---|---|---|
| **Java (Spring Boot)** | Money and state | Ledger, payment lifecycle, Kafka Streams aggregation. Anything that must be transactional and auditable. |
| **Go** | Edge and I/O concurrency | Gateway, quote service, webhook dispatcher. |
| **Python (FastAPI)** | Analytics and back office | FX ingestion, risk scoring, reconciliation, reporting, bank simulation. |
| **TypeScript / React** | Interfaces | Customer app and ops console. |

## Honest labelling

Real patterns are used throughout — transactional outbox, idempotency keys, consumer
groups, dead-letter topics, double-entry accounting. Where something is simplified for
the sake of a side project, it is labelled explicitly rather than presented as
production-ready. Those simplifications are listed in
[`05-observability.md`](05-observability.md) and
[`08-decision-log.md`](08-decision-log.md).

## Explicitly out of scope

Real payment rails. Real KYC/AML vendor integrations. Multi-tenancy. High
availability. Cloud deployment. Any regulatory compliance beyond designing as if it
mattered.
