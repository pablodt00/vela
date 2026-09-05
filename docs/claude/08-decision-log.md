# Decision log

Append as decisions are made. Format: date, decision, why, alternatives rejected.

The rejected options are the useful part. They are what stops a decision being
re-litigated six weeks later.

---

- **2026-08 — Redpanda over Kafka locally.** Lighter footprint, single binary, Kafka API
  compatible so nothing else changes. *Rejected:* full Kafka with KRaft locally.

- **2026-08 — local only, no cloud deployment.** A dead demo URL is worse than none, and
  almost all the learning is reproducible on a local cluster. *Rejected:* Oracle Cloud
  free tier (halved to 2 OCPU / 12 GB in June 2026, idle reclamation, ARM64 build
  overhead).

- **2026-09 — auth lives in `gateway`, not a separate service.** A users table, argon2
  hashes, HS256 tokens and two roles is the whole requirement. *Rejected:* Keycloak
  (~700 MB and a weekend of realm config, learning nothing about payments); a dedicated
  auth-service (a service to own one table).

- **2026-09 — quote validation is a synchronous call, not a Kafka read model.** ACCEPT
  needs an exact, current answer, and eventual consistency on the money path is the wrong
  trade. Quotes are HMAC-signed so the terms are verifiable offline. *Rejected:*
  publishing quotes to Kafka and keeping a local read model in `ledger-core`.

- **2026-09 — `webhook-dispatcher` owns endpoint registration and HMAC secrets.** It is
  the only service that needs them. *Rejected:* customer configuration in `ledger-core`.

- **2026-09 — `mock-bank` in Python.** The work is MT940/CAMT.053 generation and parsing
  plus failure injection, which is back-office string handling. *Rejected:* Go, since the
  concurrency practice is already covered by `gateway` and `webhook-dispatcher`.

- **2026-09 — `mock-bank` simulates inbound funding as well as outbound settlement.**
  Keeps the system self-contained with no external dependency, and funding then
  reconciles through the same engine as settlement. *Rejected:* Stripe test mode.

- **2026-09 — no scheduler service.** Quote expiry is lazy on read; the rest is Spring
  `@Scheduled` and Go tickers. *Rejected:* a cron-style service sweeping rows.

- **2026-09 — `exposure.snapshots` topic added, compacted.** `exposure-aggregator` had no
  defined output, so the console had no way to read exposure at all.

- **2026-09 — Redpanda's built-in Schema Registry enabled.** Zero extra containers, and
  it turns "schema changes are deliberate" from an aspiration into something CI can
  enforce. *Rejected:* leaving `contracts/` as documentation only.

- **2026-09 — analytics on Parquet + DuckDB + dbt, orchestrated by `make`.** Embedded,
  genuinely columnar, near-zero memory when idle, and dbt is the real industry tool.
  *Rejected:* Airflow (1.5 to 2 GB before doing anything, and there is no scheduling
  problem yet); ClickHouse (~1 GB, real but the GB is not available); Debezium CDC (needs
  Kafka Connect at ~1 GB, and would point at another service's database); a Postgres
  schema (free, but teaches nothing).

- **2026-09 — logs via Loki + Alloy.** ~150 MB against ~2.2 GB for ELK, which is a Java
  service that could otherwise run. Structured JSON with `correlation_id` is exactly
  Loki's shape, and Grafana is already present for metrics. *Rejected:* Elasticsearch +
  Kibana; OpenSearch + Dashboards; VictoriaLogs (lighter still, thinner Grafana
  integration).

- **2026-09 — Kafbat UI in the `kafka` profile, not `obs`.** It is a debugging tool needed
  from Phase 1, long before dashboards are. JVM heap capped at 256m, since it otherwise
  defaults to a quarter of host RAM.

- **2026-09 — tracing deferred to Phase 4.** Tempo is another ~200 MB and traces only
  become interesting once a request crosses Go to Java to Python. The Grafana
  derived-field config is written and commented out, ready.

- **2026-09 — `customer-app` polls; no websockets.** Connection state and reconnection
  handling to save one request every few seconds for one user is over-engineering.

- **2026-09 — `webhook-dispatcher` kept despite having no machine customers.** The
  delivery itself is not the point; retry with exponential backoff, HMAC signing and
  dead-lettering are a real distributed-systems problem and are Go doing what Go is good
  at. The test receiver is a fifty-line stub, not a second application.
