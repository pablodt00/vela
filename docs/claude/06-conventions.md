# Conventions

## Repo layout

```
vela/
  services/
    gateway/            (Go)
    quote-service/      (Go)
    webhook-dispatcher/ (Go)
    ledger-core/        (Java)
    exposure-aggregator/(Java)
    fx-ingester/        (Python)
    risk-service/       (Python)
    recon-service/      (Python)
    mock-bank/          (Python)
    analytics-sink/     (Python)
    analytics-api/      (Python)
  apps/
    customer-app/       (React)
    ops-console/        (React)
  analytics/            (dbt-duckdb project)
  manifests/            (k8s YAML, applied to a local k3d cluster)
  contracts/            (Avro or JSON Schema for every Kafka topic)
  observability/        (Prometheus, Loki, Alloy, Grafana config)
  docs/
  compose.yaml
  compose.obs.yaml
```

`contracts/` is the source of truth for event shapes. A schema change is a deliberate,
versioned act — never an incidental edit inside one service. Schemas are registered with
Redpanda's built-in Schema Registry, which is what makes compatibility checkable in CI.

## Events

Every event carries: `event_id` (UUID), `event_type`, `occurred_at`, `version`,
`correlation_id`, and a `payload`. Consumers dedupe on `event_id`.

## Errors

API errors return RFC 7807 `problem+json`. Never leak internal exception messages to the
customer app; the ops console may see more detail.

## Idempotency

Every mutating API endpoint accepts an `Idempotency-Key` header. `gateway` enforces its
presence; `ledger-core` enforces its semantics.

Every Kafka consumer is idempotent. Assume every message is delivered more than once.

## Money

Never a float. Integer minor units, with an ISO 4217 currency code alongside, always.

## Naming

- Topics: `domain.event-type`, lowercase, dot-separated.
- Services: kebab-case.
- Java packages: `com.vela.<service>`.
- Go modules: `github.com/<user>/vela/services/<service>`.

Code, identifiers, commit messages, file names, docs and diagrams are all in English.

## Commits

Conventional commits, scoped by service: `feat(ledger-core): add outbox relay`.

## Testing

Business logic gets real tests: ledger invariants, reconciliation matching, risk rules.
CRUD glue does not.

## Databases

No shared databases between services. Cross-service reads go through Kafka or an API.
One Postgres container with a database per service — the separation is enforced by
convention and review, not by separate instances.

## Definition of done for any service

- Dockerfile
- k8s manifest, even though deployment is local
- `/health` and `/metrics` endpoints
- Structured JSON logging including `correlation_id`
- README explaining what it owns
- Tests on its business logic

## JVM services

Explicit heap cap, always. `-Xmx512m`. Never rely on JVM defaults.
