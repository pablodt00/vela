# Domain model

## Payment lifecycle

```
QUOTED -> ACCEPTED -> FUNDED -> CONVERTED -> SETTLING -> SETTLED
              |          |           |
          REJECTED   REJECTED     FAILED -> REVERSED
```

- **QUOTED** — quote issued with a rate, spread and TTL. No ledger impact.
- **ACCEPTED** — customer commits. Risk check fires. Rate is locked.
- **FUNDED** — source funds received. Simulated end to end by `mock-bank`, which emits
  an inbound credit on `bank.statements` that `recon-service` matches and `ledger-core`
  acts on. No external payment provider is involved.
- **CONVERTED** — FX conversion booked. Ledger entries written.
- **SETTLING** — payout instruction sent to `mock-bank`.
- **SETTLED** — confirmed by bank statement, matched by `recon-service`.
- **FAILED / REVERSED** — settlement failed; compensating ledger entries written. The
  original entries are never deleted.

State transitions live in `ledger-core` and only there. Other services react to events;
they do not mutate payment state.

## Ledger model

Double-entry, append-only. Two tables.

**accounts** — `id`, `type`, `currency`, `owner_id`

Account types: `customer_balance`, `fx_position`, `fee_income`, `bank_settlement`,
`suspense`.

**entries** — `id`, `transaction_id`, `account_id`, `direction` (debit/credit),
`amount_minor`, `currency`, `created_at`, `payment_id`

Rules:

- Entries are immutable. Corrections are new compensating entries, never updates.
- All entries sharing a `transaction_id` are written in a single database transaction
  and must sum to zero per currency.
- Balances are derived from entries. A cached balance is an optimization, never the
  truth.

### Example: EUR to GBP conversion of 1000 EUR at 0.8500, 0.5% spread

```
txn_1
  debit   customer_balance EUR   100000
  credit  fx_position      EUR   100000
  debit   fx_position      GBP    84575
  credit  customer_balance GBP    84575
  debit   fx_position      EUR      500   (spread)
  credit  fee_income       EUR      500
```

Amounts are integer minor units. 100000 is 1000.00 EUR.

## Reconciliation

`recon-service` consumes `bank.statements` and matches statement lines against
`ledger.entries` on the `bank_settlement` accounts.

Inbound funding credits reconcile through the same path as outbound settlement — same
matching logic, same break categories. This is deliberate: one reconciliation engine,
exercised twice per payment.

Match categories:

- **Matched** — amount, currency, date and reference all align.
- **Break: missing in bank** — ledger says settled, no statement line. Usually timing.
- **Break: missing in ledger** — statement line with no ledger entry. Unexpected credit;
  it goes to a suspense account until it can be attributed.
- **Break: amount mismatch** — partial settlement, or a fee deducted by the
  correspondent bank in transit.
- **Duplicate** — the same statement line seen twice. Must not double-post.

Breaks land in a queue in the ops console with an assignable status. Auto-matching
handles the clean cases; the interesting engineering is in the fuzzy ones — amount
within tolerance, reference partially matching, settlement date drift.

## Risk scoring

`risk-service` consumes `payments.events` at ACCEPTED and emits a decision within a
deadline. If it does not answer in time, `ledger-core` treats the payment as `review`
and holds it. Silence never means approval.

Signals: velocity per customer, amount versus that customer's history, new beneficiary,
unusual corridor, round-number amounts, time of day, rapid quote-accept cycles.

Output: `approve` | `review` | `block`, plus a score and the contributing reasons. The
reasons matter — an unexplained score is unusable in an ops console.

## Glossary

The banking vocabulary used throughout this repository.

- **Corridor** — a currency/country pair being sent between, e.g. EUR to GBP.
- **Spread** — the margin added to the market rate. The platform's revenue. If the
  market says 1 EUR = 0.8500 GBP and the platform quotes 0.8458, the difference is the
  spread. There are no separate fees on top.
- **Exposure / position** — the net open amount in a currency that the platform holds
  and is exposed to rate movement on. Sell a customer GBP for euros and the platform is
  holding a GBP position until it is hedged.
- **Break** — an unmatched item in reconciliation.
- **Suspense account** — where money sits when it cannot yet be attributed correctly.
- **Value date** — the date funds are actually available, distinct from the booking date.
- **MT940 / CAMT.053** — standard bank statement formats. A statement is a list of
  lines: date, amount, currency, reference. `mock-bank` emits both.
- **Partial settlement** — 1000 was sent, 995 arrived, because a correspondent bank
  took a fee in transit.
- **Outbox** — a table written in the same transaction as the business data, later
  relayed to Kafka. Prevents the "database committed but event lost" failure.
- **Double entry** — the accounting rule that every movement is recorded twice, once as
  a debit and once as a credit, summing to zero. Money leaving one place must arrive
  somewhere.
- **Idempotency key** — a unique ID the client generates per operation, so a retry after
  a dropped connection converts money once rather than twice.
