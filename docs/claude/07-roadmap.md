# Roadmap

## Current status

**Phase: 0 — not started.**

Update this line whenever a phase completes. It is the first thing to read to know where
things stand.

---

## Phase 0 — skeleton

Monorepo, docker-compose (Redpanda + Postgres) with per-service profiles, a `/health`
endpoint in each of Java, Go and Python, GitHub Actions building all images.

**Done when:** `docker compose up` brings up three services in three languages and CI is
green.

## Phase 1 — vertical slice

`fx-ingester` publishes to `fx.rates`. `ledger-core` accepts a conversion request, writes
balanced double-entry rows, emits `ledger.entries` via the outbox.

**Done when:** a curl request produces a balanced ledger transaction and an event on
Kafka.

> **Open decision.** With funding now going through `mock-bank`, `recon-service` sits on
> the happy path rather than being only an ops tool — so this slice cannot reach FUNDED
> without it. Either stop this phase at CONVERTED with funding faked by a direct API
> call, or pull a minimal `mock-bank` forward into Phase 1. Faking it here and keeping the
> phases as they are is the lighter option. Decide before starting, not mid-slice.

## Phase 2 — the platform layer, locally

Write k8s manifests for everything built so far, stand up k3d, point ArgoCD at the
`manifests/` directory. Practise the GitOps loop without a cloud bill.

**Done when:** a commit to `manifests/` changes what is running in the local cluster,
unattended.

## Phase 3 — the fintech meat

`mock-bank`, `recon-service`, `risk-service`. Statement generation with injected
failures.

**Done when:** an injected duplicate credit and a partial settlement both surface as
correctly categorized breaks, and an inbound funding credit moves a payment to FUNDED.

## Phase 4 — Go at the edge

`gateway` (auth, rate limiting, idempotency keys), `webhook-dispatcher` (retries, HMAC),
`quote-service` with Redis cache.

**Done when:** a customer-facing webhook survives a receiver being down for five minutes.

Tempo and distributed tracing become worthwhile here, once a request crosses Go to Java
to Python.

## Phase 5 — frontends

`ops-console` first, then `customer-app`.

**Done when:** a break can be investigated and resolved entirely from the console.

## Phase 5.5 — the data layer

`analytics-sink`, the dbt-duckdb project, `analytics-api`, and a reporting tab in the
console.

Deliberately not earlier: before Phase 3 there are only three event types and no
interesting questions to ask of them. The sink can be written sooner and left running to
accumulate history.

**Done when:** the console shows volume by corridor and break ageing, computed from
Parquet rather than from the operational database.

## Phase 6 — platform polish

OpenTelemetry across all three languages, Prometheus + Grafana, load test, deliberate
chaos — kill a consumer mid-batch and prove nothing double-counts.

## Phase 7 — the artifact (replaces deploying)

Since nothing is publicly hosted, the deliverable is a strong README with the
architecture diagram, a `make demo` that seeds data and runs an end-to-end payment, and a
short screen recording of the ops console resolving a reconciliation break.

**Done when:** someone can clone the repo and see the system work in under ten minutes.

> Test the `demo` compose profile early. It has to actually fit in memory on the target
> laptop, and finding that out in Phase 7 is finding out too late.
