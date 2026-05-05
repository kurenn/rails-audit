# Dimension: Money paths

Payments, payouts, refunds, commissions — anything that moves real money or external state with financial consequence. Defaults to **blocker** when defective.

## What "good" looks like

- Money stored as `Decimal` (or `bigint` cents) — never `Float`.
- A column-level convention is documented (e.g. all amounts are `*_cents` integers in the smallest currency unit).
- Stripe (or other PSP) calls go through a single wrapper service — never direct API calls in controllers.
- Every external mutation (charge, transfer, payout, refund) sends an **idempotency key** derived from a stable internal identifier.
- Webhooks are signature-verified AND event-ID-deduplicated.
- Multi-step money operations are wrapped in `ActiveRecord::Base.transaction` with the external API call placed *after* the local DB write commits, OR with a saga/outbox pattern for crash safety.
- Refunds and reversals have their own idempotency.
- Currency conversion (if multi-currency) goes through a single source of truth, with FX rates snapshotted on the transaction record.
- Audit trail: every money-moving record has `created_by_user_id`, `processed_at`, `external_reference_id` (e.g. Stripe charge ID).

## Checks

### Column types

```bash
# Read schema and grep for money-shaped columns
grep -E "\.(decimal|float|integer|bigint).*(amount|price|cost|fee|commission|payout|charge|balance|cents)" db/schema.rb
```

Any `t.float :amount` / `:price` / etc. → **Blocker** (silent rounding).

### Service layer wrapping

```bash
ls app/services/payments/ app/services/stripe/ 2>/dev/null
# Direct Stripe calls in controllers
grep -rn "Stripe::" app/controllers/ --include="*.rb"
# Direct in views/serializers (worse)
grep -rn "Stripe::" app/views/ app/serializers/ 2>/dev/null
```

If `Stripe::*` calls appear in controllers, they should be small (probably account-info reads). Any mutation (`Stripe::Charge.create`, `Stripe::Transfer.create`, `Stripe::Payout.create`, `PaymentIntent.create`) in a controller → **High** (move to service layer).

### Idempotency keys

```bash
# Stripe Ruby SDK supports idempotency_key as a header on most create calls
grep -rn "Stripe::" app/services/ app/jobs/ --include="*.rb" -A 5 | grep -E "idempotency_key|Idempotency-Key"
# vs total Stripe mutations
grep -rn "Stripe::.*\(create\|update\|capture\|refund\)" app/services/ app/jobs/ --include="*.rb"
```

Calculate the ratio. Mutations without idempotency keys → **High** each. Concentrations → **Blocker**.

A correct idempotency key is derived from a *stable* internal ID, not `SecureRandom` or `Time.now`. Look for the pattern `idempotency_key: "payout-#{payout.id}"` or similar.

### Webhook event dedup

```bash
# Look for a processed_events / stripe_events table
grep -E "create_table.*event|create_table.*webhook" db/schema.rb
# Look for the dedup check in webhook controller
grep -rn "find_by.*event_id\|find_or_create.*event\|StripeEvent\|ProcessedEvent" app/controllers/webhooks/ app/services/ 2>/dev/null
```

No dedup table → **High**. The check needs to be transactional with the side-effect: `INSERT ... ON CONFLICT DO NOTHING` or wrap in a transaction with row-level lock.

### Transactional boundaries

For each money-moving operation, the pattern should be:

```ruby
# Good
Payout.transaction do
  payout = Payout.create!(amount_cents: amount, status: :pending, ...)
  result = Stripe::Payout.create({...}, idempotency_key: "payout-#{payout.id}")
  payout.update!(stripe_payout_id: result.id, status: :submitted)
end
```

Anti-patterns:
- DB write *after* external call (if external succeeds + DB fails → orphaned external state).
- External call *inside* transaction without explicit handling (long-held DB locks).
- No record created before the external call (no idempotency anchor).

Read each service file under `app/services/payments/` or wherever payment logic lives. For each, document the order: DB-record-create → external-call → DB-record-update? Or some other sequence?

### Currency / unit confusion

```bash
# Look for inconsistent units
grep -rnE "(amount|fee|price)\s*\*\s*100" app/ --include="*.rb"  # likely "convert to cents"
grep -rnE "(amount|fee|price)\s*/\s*100" app/ --include="*.rb"  # likely "convert from cents"
# These are fine but should be centralized in one helper, not scattered
```

Multiple `* 100` / `/ 100` sites → **Medium** (centralize in a `Money` value object or use the `money-rails` gem).

### Negative / overflow / precision

```bash
# Find the entry points where amounts come from clients
grep -rnE "params\[:amount|params\[:price|params\[:cost" app/controllers/ --include="*.rb"
# Look at validation
```

Each entry point needs:
- Type coercion (`to_i` is a smell — fail loudly on non-numeric)
- Min check (no negative, no zero unless explicitly allowed)
- Max check (no overflow, no impossible amounts)
- Decimal precision check if Decimal columns

### Refund / reversal idempotency

```bash
grep -rn "Stripe::Refund\|refund\b" app/services/payments/ 2>/dev/null
```

Refund operations are bug magnets — partial refunds, double refunds, refund-then-fail-to-record. Each needs its own idempotency key and audit row.

### Audit trail

For each money-moving table (`transactions`, `payouts`, `payments`, `commissions`):

```bash
# Schema columns
grep -A 20 "create_table.*<table>" db/schema.rb
```

Required columns:
- `external_reference_id` (Stripe charge/transfer/payout ID)
- `status` (with explicit state machine, ideally `aasm` or similar)
- `created_at` / `updated_at`
- `processed_at` (separate from `created_at`)
- `error_code` / `error_message` for failures
- `idempotency_key` (the one you sent — for debugging)

### Background job money safety

If any payment is processed in a background job, the job needs:
- Idempotency at the *job* level (re-running the job N times = same result)
- Explicit retry config (not blanket — define which exceptions retry vs which discard)
- A way to surface stuck jobs (e.g. job > N minutes old in pending state alerts)

Cross-reference with `dimensions/background-jobs.md`.

### Test coverage on money paths

This dimension overlaps with coverage. Money services should be at **≥85% line, ≥75% branch** coverage with explicit specs for:
- Successful happy path
- External API failure (Stripe `CardError`, `RateLimitError`, network timeout)
- Webhook replay
- Concurrent identical request (idempotency in action)
- Refund of a non-existent or already-refunded charge
- Currency rounding edge cases (1 cent, 0 cents, max int)

If money services have <50% coverage → escalate to **Blocker** in this dimension regardless of other metrics.

## Cross-cuts

Money findings often touch multiple dimensions. Default secondary tags:

- **`background-jobs`** — payment jobs without retry/idempotency are both jobs *and* money issues.
- **`reliability`** — missing timeouts on Stripe SDK, missing circuit breakers cross with reliability.
- **`security-and-authz`** — webhook signature verification is security primary; money secondary.
- **`data-integrity`** — multi-step money writes without `transaction do…end` cross with data integrity.
- **`code-smells`** — `update_column` on `stripe_*` columns is a smell *and* money-relevant.
- **`observability`** — money operations without audit trail or APM tracing surface here too.

## Severity calibration

| Pattern | Default tier |
|---|---|
| `t.float :amount` (or any money-as-Float) | Blocker |
| Stripe webhook without event-ID dedup | High (Blocker if recent commits added new event types) |
| Stripe mutation without idempotency key | High |
| External API call with DB write *after* (orphan risk) | Blocker |
| Refund/reversal logic without idempotency | Blocker |
| Direct `Stripe::*` mutations in controller | High |
| `params[:amount].to_i` without bounds | High |
| No audit columns on money table | High |
| Currency conversion scattered (`* 100` everywhere) | Medium |
| Money services at <50% coverage | Blocker |
| `Money.from_cents(amount)` style (with money gem) | Good — flag if absent on multi-currency |
