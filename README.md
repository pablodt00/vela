# vela
Vela is a portfolio-grade simulation of a cross-border payments and FX platform.

A customer requests a quote to convert one currency into another, accepts it, funds it, and the platform converts, settles and reconciles the payment. Market data is real where free sources allow. The banking rails are simulated.

Vela does not move real money and never will — no licence, no real rails. Everything above the rails is built for real: ledger correctness, reconciliation, risk scoring, exposure aggregation, operational tooling.

## .gitignore
File adapted for Python, Java, Go and Node.

## Directories
### analytics
dbt-duckdb project

### apps
#### customer-app
Quote to convert to balances to payment history.

#### ops-console
Payment inspection, risk queue, reconciliation breaks, exposure dashboard.

### contracts
Avro or JSON Schema for every Kafka topic.

### docs
Useful docs for the project.

### manifests
k8s YAML, applied to a local k3d cluster.

### observability
Prometheus, Loki, Alloy, Grafana config.

### services
#### analytics-api
Read-only queries over DuckDB for the ops console reporting tab.

#### analytics-sink
Consumes every topic, writes date-partitioned Parquet.

#### exposure-aggregator
Windowed net exposure per currency pair, published for the ops console.

#### fx-ingester
Pulls ECB reference rates and crypto websocket ticks, normalizes, publishes to `fx.rates`.

#### gateway
Auth (issues and validates JWTs, owns users and roles), rate limiting, request routing, idempotency-key enforcement at the edge.

#### ledger-core
Double-entry ledger, payment state machine, transactional outbox.

#### mock-bank
Simulates both directions: inbound funding confirmations and outbound settlement. Injects delays, partial settlements, duplicates, out-of-order statements.

#### quote-service
Serves FX quotes from cached rates, applies spread, issues quote IDs with a TTL, HMAC-signs quote terms.

#### recon-service
Parses mock bank statements, matches against ledger entries, surfaces breaks.

#### risk-service
Scores payments against rules and a simple model; emits decisions (`approve`, `review`, `block`)

#### webhook-dispatcher
Owns customer endpoint registration and per-customer HMAC secrets. Delivers events to those endpoints; retries with exponential backoff, dead-letters after N attempts.
