# Services, in plain language

What each service does, and the banking concepts behind it. This is the file to read if
the domain vocabulary is unfamiliar.

---

## `gateway` (Go)

The front door. Every request from a browser or from a customer's server hits this
first.

- **Issues and validates JWTs.** A JWT is a signed token proving "this request is from
  customer 42". The gateway hands one out at login and checks it on every later request,
  so no downstream service has to think about passwords.
- **Rate limiting.** Caps how many requests a customer can make per minute, so one
  client cannot take the platform down.
- **Routing.** Forwards each request to whichever service owns it.
- **Enforces the `Idempotency-Key` header.** If a client's network drops and they retry,
  the same key arrives twice; that is how the platform knows to convert 1000 EUR once,
  not twice. The gateway only checks the header is *present*. Whether it has been seen
  before is `ledger-core`'s job.

Auth lives here rather than in a separate service: a users table, argon2 hashes, HS256
tokens and two roles is the entire requirement.

---

## `quote-service` (Go)

Answers "what rate would I get for EUR to GBP right now?"

- **Serves quotes from cached rates.** Reads the latest market rate out of Redis rather
  than recalculating per request.
- **Applies the spread.** The margin over the market rate. That margin *is* the revenue.
- **Gives each quote a TTL** — typically 30 to 60 seconds. Rates move, so a quote cannot
  be honoured forever. Accept it after it expires and it is refused.
- **HMAC-signs the quote terms.** A signature over the rate, spread and expiry, letting
  `ledger-core` verify the numbers it was handed were not tampered with.

---

## `webhook-dispatcher` (Go)

Pushes events to customers' own servers so they do not have to poll.

- **Stores each customer's endpoint URL and secret.** No other service needs these, so
  it owns them.
- **Delivers events** — "your payment settled" as an HTTP POST.
- **Signs each delivery with HMAC** — a hash of the body plus the shared secret, in a
  header. The customer recomputes it to prove the call genuinely came from Vela and the
  payload was not altered.
- **Retries with exponential backoff.** Receiver down? Retry after 1s, then 2s, 4s, 8s.
  Backing off avoids hammering a server that is already struggling.
- **Dead-letters after N attempts.** Gives up and parks the event on a `.dlq` topic for a
  human, rather than retrying forever.

**Why this exists when there is a customer app:** webhooks are not for browsers. They
are for the other kind of customer — a business integrating Vela into its own backend,
whose server needs to know hours later that a payment settled. That is the standard
model for payments platforms: a dashboard for humans, an API plus webhooks for machines.

This is the least strictly necessary service in the project, since Vela has no machine
customers. It is kept because the engineering in it — retry with backoff, HMAC signing,
dead-lettering — is a real distributed-systems problem and is Go doing what Go is good
at. The receiver used to test it should be a fifty-line stub, not a second application.

---

## `ledger-core` (Java / Spring Boot)

The important one. The only service allowed to say what a payment's state is, or to move
money.

- **Runs the payment state machine.** QUOTED, ACCEPTED, FUNDED, CONVERTED, SETTLING,
  SETTLED. Every other service *reacts* to these transitions; none of them cause one
  directly.
- **Writes double-entry ledger rows.** Every movement is recorded twice, once as a debit
  and once as a credit, and they must sum to zero per currency. If a transaction does not
  balance it is a bug, and the ledger refuses to write it. This is the invariant the whole
  project exists to get right.
- **Never updates or deletes an entry.** A mistake is fixed by writing an opposite entry
  alongside it, not by editing history. Auditors need to see what happened *and* what
  corrected it.
- **Publishes via a transactional outbox.** The failure this prevents: the database
  commits, then the process dies before the Kafka event is sent. Database updated, nobody
  told. So the event is written into an `outbox` table *in the same database transaction*
  as the business data — both commit or neither does — and a separate relay reads that
  table and publishes. No dual writes.
- **Enforces idempotency semantics.** The same `Idempotency-Key` seen twice returns the
  original result rather than converting again.
- **Calls `quote-service` at ACCEPT** to confirm the quote is real, unused and unexpired.
- **Holds on missing risk decisions.** No answer within the deadline is treated as
  `review`.

Heap capped at `-Xmx512m`.

---

## `exposure-aggregator` (Java / Kafka Streams)

Tracks how much of the platform's own money is at risk.

- **Computes net exposure per currency pair.** Sell customers 100k GBP for euros and the
  platform is holding a GBP position. If GBP moves before that is hedged, the platform
  gains or loses. Real FX desks watch this constantly.
- **Windowed** — recalculated over rolling time buckets rather than from all history.
- **Publishes `exposure.snapshots`** — compacted, so the ops console reads the current
  figure per pair instantly.

Heap capped at `-Xmx512m`.

---

## `fx-ingester` (Python / FastAPI)

Where market rates come from.

- **Pulls ECB reference rates.** The European Central Bank publishes official daily
  rates, free, no key required.
- **Reads crypto websocket ticks.** A live price stream, so there is something moving
  second by second to work with.
- **Normalizes them.** Different sources, different formats, one internal shape.
- **Publishes to `fx.rates`,** feeding `quote-service`, exposure and the console.

---

## `risk-service` (Python)

Decides whether a payment looks like fraud.

- **Consumes payments at ACCEPTED** and scores them.
- **Signals:** velocity (ten payments in an hour from someone who normally sends one a
  month), amount versus that customer's history, a beneficiary never paid before, an
  unusual corridor, suspiciously round amounts, odd hours, accepting quotes unnaturally
  fast.
- **Emits `approve` | `review` | `block`** with a score *and the reasons*. An unexplained
  number is useless to whoever works the queue; they need "new beneficiary plus five
  times the usual amount".
- **Must answer within a deadline.** Silence is treated as `review` by `ledger-core`.

---

## `mock-bank` (Python)

Stands in for the real banking rails, which Vela will never touch.

- **Confirms inbound funding** — the customer's money arriving at the platform.
- **Executes outbound settlement** — the payout going to the beneficiary.
- **Emits bank statements** in MT940 and CAMT.053, the formats banks actually use.
- **Injects realistic failures on purpose:** delays, partial settlements, duplicate
  lines, statements arriving out of order. Real rails do all of this, and handling it is
  the interesting engineering.

Python rather than Go: the work is statement generation and parsing plus failure
injection, which is back-office string handling. The concurrency practice Go would offer
is already covered by `gateway` and `webhook-dispatcher`.

---

## `recon-service` (Python)

Reconciliation: proving the platform's records match the bank's.

- **Reads `bank.statements`** and matches each line against ledger entries.
- **Matched** — amount, currency, date and reference all line up. Most lines.
- **Break: missing in bank** — the ledger says settled, the statement has nothing. A
  *break* is simply an unmatched item. Usually timing; the money is in flight.
- **Break: missing in ledger** — a statement line with no ledger entry. Unexpected money,
  parked in a suspense account until it can be attributed.
- **Break: amount mismatch** — a partial settlement, or a fee taken en route.
- **Duplicate** — the same line twice. Must not post twice, or the ledger invents money.
- **The hard part is fuzzy matching:** amount within tolerance, reference only partially
  matching, settlement date drifting by a day. Exact matches are trivial; these are where
  the work is.
- **Breaks go to a queue** in the ops console for someone to resolve.

Because `mock-bank` also simulates inbound funding, this service sits on the happy path,
not only in operations.

---

## `analytics-sink` and `analytics-api` (Python)

The reporting layer, kept off the operational path.

- **Sink** — consumes every topic and writes date-partitioned Parquet files. Parquet is a
  columnar format: fast for "sum all volume by corridor", slow for "fetch this one
  payment", which is the opposite of Postgres and exactly right for reporting.
- **A dbt project** (`analytics/`, using dbt-duckdb) turns those raw files into clean
  models: volume by corridor, spread revenue, break ageing and resolution time, risk
  decision distribution versus outcomes, position history.
- **API** — read-only DuckDB queries for the console's reporting tab.

Two rules: the warehouse is **derived, never truth** — if a mart disagrees with the
ledger, the mart is wrong — and it reads from Kafka, never from another service's
Postgres.

Orchestration is `make` targets, not Airflow. There is no scheduling problem yet, and
Airflow alone would cost 1.5 to 2 GB before doing anything.

---

## `ops-console` (React)

The internal tool, and realistically the most impressive thing in the repository,
because it is what an operations team would actually use.

- Inspect a payment end to end
- Work the risk review queue, approving or blocking with the reasons visible
- Investigate and resolve reconciliation breaks
- Watch live FX exposure
- Reporting tab, backed by `analytics-api`

---

## `customer-app` (React)

The customer-facing side: get a quote, accept it, see balances, see payment history.
Deliberately last — it is the least interesting engineering in the project.

It polls for payment status. A websocket layer would be over-engineering: connection
state and reconnection handling, to save one request every few seconds for one user.
