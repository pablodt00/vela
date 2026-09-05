# Architecture

## System summary

A customer requests a quote for a currency conversion, accepts it, funds it, and the
platform converts, settles and reconciles. Rails are simulated by `mock-bank`, in both
directions — inbound funding and outbound settlement. Market data is real where free
sources allow.

## Services

| Service | Language | Responsibility |
|---|---|---|
| `gateway` | Go | Auth (issues and validates JWTs, owns users and roles), rate limiting, request routing, idempotency-key enforcement at the edge |
| `quote-service` | Go | Serves FX quotes from cached rates, applies spread, issues quote IDs with a TTL, HMAC-signs quote terms |
| `webhook-dispatcher` | Go | Owns customer endpoint registration and per-customer HMAC secrets. Delivers events to those endpoints; retries with exponential backoff, dead-letters after N attempts |
| `ledger-core` | Java / Spring Boot | Double-entry ledger, payment state machine, transactional outbox |
| `exposure-aggregator` | Java / Kafka Streams | Windowed net exposure per currency pair, published for the ops console |
| `fx-ingester` | Python / FastAPI | Pulls ECB reference rates and crypto websocket ticks, normalizes, publishes to `fx.rates` |
| `risk-service` | Python | Scores payments against rules and a simple model; emits decisions (`approve`, `review`, `block`) |
| `recon-service` | Python | Parses mock bank statements, matches against ledger entries, surfaces breaks |
| `mock-bank` | Python | Simulates both directions: inbound funding confirmations and outbound settlement. Injects delays, partial settlements, duplicates, out-of-order statements |
| `analytics-sink` | Python | Consumes every topic, writes date-partitioned Parquet |
| `analytics-api` | Python / FastAPI | Read-only queries over DuckDB for the ops console reporting tab |
| `customer-app` | TypeScript / React | Quote to convert to balances to payment history |
| `ops-console` | TypeScript / React | Payment inspection, risk queue, reconciliation breaks, exposure dashboard |

## Kafka topics

| Topic | Producer | Key | Consumers |
|---|---|---|---|
| `fx.rates` | fx-ingester | currency pair | quote-service, exposure-aggregator, ops-console feed, analytics-sink |
| `payments.events` | ledger-core (via outbox) | payment ID | risk-service, webhook-dispatcher, recon-service, analytics-sink |
| `ledger.entries` | ledger-core (via outbox) | account ID | exposure-aggregator, recon-service, analytics-sink |
| `risk.decisions` | risk-service | payment ID | ledger-core, ops-console feed, analytics-sink |
| `bank.statements` | mock-bank | account ID | recon-service, analytics-sink |
| `exposure.snapshots` | exposure-aggregator | currency pair | ops-console feed |

Partition by the key above so per-entity ordering holds. Dead-letter topics get a
`.dlq` suffix. `exposure.snapshots` is compacted, so the console can read the current
exposure per pair without replaying a window.

## Non-negotiable invariants

1. Every ledger transaction balances: debits equal credits, per currency, always.
2. No service reads another service's database. Ever.
3. Every Kafka consumer is idempotent — dedupe on event ID, or make the operation
   naturally idempotent. Assume every message is delivered more than once.
4. `ledger-core` publishes only via the transactional outbox. No dual writes.
5. Amounts are integer minor units plus an ISO 4217 currency code. No floats, no
   implicit currency.
6. Quotes expire. An accepted quote past its TTL is rejected, not honoured.

## Kafka or an API?

Kafka is the default for anything that is a fact about something that already
happened, consumed by parties the producer does not need to know about.

A synchronous API call is correct when the caller needs an answer now and needs it to
be exact — read-your-writes, no eventual consistency. Two such cases exist:

- `ledger-core` calls `quote-service` to validate a quote at ACCEPT.
- `ops-console` calls service APIs for on-demand reads.

Reaching for Kafka where the caller must block on an exact answer is over-engineering,
not decoupling.

### Quote validation

`ledger-core` calls `quote-service` at ACCEPT to confirm the quote exists, is unused
and is within its TTL. Quotes are additionally HMAC-signed by `quote-service` over
`{quote_id, rate, spread, expires_at}`, so `ledger-core` can verify that the terms it
was handed were not tampered with, independently of the call.

### Scheduling

There is no scheduler service and there will not be one. Quote expiry is evaluated
lazily on read, never by a job sweeping rows. The risk decision deadline and recon runs
use Spring `@Scheduled`; webhook retry timing uses a ticker in Go.

## Data stores

One Postgres container, one database per service. Services never query another
service's database — the separation is enforced by convention and by review, not by
separate instances. Redis sits in front of `quote-service` for hot rate lookups.

Redpanda's built-in Schema Registry is enabled (port 8081, no extra container).
`contracts/` remains the source of truth for event shapes; the registry is what makes
that enforceable, with compatibility checks in CI.

The analytics layer stores date-partitioned Parquet files on disk, queried through
DuckDB. It is derived data: if a mart disagrees with the ledger, the ledger is right
and the mart is wrong.

## Runtime environment — local only

**docker-compose (default, day to day).** Redpanda (single broker), Postgres, Redis,
plus the services under edit. Profiles bring up a subset — for example
`compose --profile ledger up` starts Redpanda, Postgres and `ledger-core` only.

**k3d or kind (when working on the platform layer).** A local Kubernetes cluster from
the same `manifests/` directory, with ArgoCD pointed at a local path or the GitHub
repo. Used to validate manifests and practise the GitOps loop, not to serve traffic.

## Memory budget

Rough working figures per container.

| Component | Resident |
|---|---|
| Redpanda | ~1 GB |
| Postgres | ~256 MB |
| Redis | ~50 MB |
| Each Java service (`-Xmx512m`) | ~700 MB |
| Each Go service | ~50 MB |
| Each Python service | ~150 MB |
| Kafbat UI (JVM, `-Xmx256m`) | ~350 MB |
| Observability stack (Prometheus, Grafana, Loki, Alloy) | ~630 MB |

**Everything at once is roughly 5.5 GB before an IDE, a browser and the OS.** That is
not a machine anyone here owns. The service list is a catalogue of profiles, not a
system to run in one go.

### Practical profiles

| Profile | Contents | Approx. |
|---|---|---|
| `ledger` | Redpanda, Postgres, `ledger-core` | ~2 GB |
| `recon` | Redpanda, Postgres, `mock-bank`, `recon-service` | ~1.6 GB |
| `edge` | Redpanda, Redis, `gateway`, `quote-service` | ~1.2 GB |
| `demo` | Everything except `exposure-aggregator`, analytics and observability | ~2.9 GB |

The `demo` profile is the one to test early — Phase 7's `make demo` has to actually run
on the target laptop.

## Explicitly out of scope

Real payment rails, real KYC/AML vendor integrations, multi-tenancy, high availability,
cloud deployment, and any regulatory compliance beyond designing as if it mattered.
