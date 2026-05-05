# Money-path re-validation

A focused second pass that re-checks every finding tagged with `money-and-payments` (in `primary_dimension` OR `secondary_dimensions[]`) against `dimensions/money-and-payments.md` specifically. Catches what general-purpose synthesis misses: idempotency-key derivation patterns, transaction-boundary ordering, refund/reversal idempotency, currency-precision issues, and audit-trail completeness.

Runs as **Step 4.5** in `SKILL.md` — *after* synthesis (Step 4) produces the JSON, *before* self-check (Step 5.5). Re-validation may **refine evidence**, **reject false positives**, and **promote severity** when a money-specific check applies. It will not silently drop findings without recording a `status: rejected` and `notes`.

---

## Why a second pass

The 4 cluster agents in Step 3 each cover a slice. Cluster D (security & money) has the money dimension file — but Cluster D agents are time-bounded and cover security + AuthZ + money + governance. Money-specific patterns get under-checked when the agent is also racing to cover SSRF, JWT, IDOR.

A focused pass with **just** the money-and-payments dimension file in scope catches things the cluster missed:

- Stripe mutations missing `idempotency_key:` derived from a stable internal ID
- Multi-step money writes where the external API call lands *before* the local DB write commits (orphan-state risk)
- Refund/reversal logic without its own idempotency key
- Webhook handlers without an event-ID dedup table
- Money columns typed as `Float` instead of `Decimal` or `bigint cents`
- Currency conversion sites scattered across files (`* 100` / `/ 100`) without a centralized helper
- Audit columns missing on money-moving tables (`processed_at`, `external_reference_id`, `idempotency_key`, `status` with explicit state machine)

---

## Scope

Re-validation operates on the set:

```
revalidation_set = findings.select { |f|
  f.primary_dimension == "money-and-payments" ||
  f.secondary_dimensions.include?("money-and-payments")
}
```

Plus a **proactive sweep**: read `app/services/payments/*`, `app/controllers/webhooks/stripe_*`, `app/jobs/*payment*` and `app/jobs/*payout*`, and any file with money-shaped column names in `db/schema.rb`. New findings discovered in the sweep are added to `findings[]` with `tool_origin: "money-revalidation"`.

---

## Statuses

For each finding in the revalidation set, set `findings[<id>].money_revalidation`:

| Status | When |
|---|---|
| `confirmed` | Finding stands as written; evidence verified against the money dimension's checks |
| `refined` | Same finding, but evidence/explanation/fix_sketch is improved with money-specific detail |
| `rejected` | False positive on closer look — finding is removed from the punch list (kept in JSON with `ignored: true` and a `notes` explaining why) |
| `promoted` | Severity bumped one tier (medium → high, high → blocker) because a money-specific check applies that the original synthesis missed |

`notes` is required for `refined`, `rejected`, and `promoted` — explain what the money-specific check revealed.

---

## The re-validation prompt

When running re-validation as a focused inline check (or via a focused subagent), use this prompt template:

> You are re-validating a Rails-audit finding against money-and-payments-specific checks. The finding is below. The full money-and-payments dimension file is provided. The cited file's relevant region is provided.
>
> Decide: **confirmed** / **refined** / **rejected** / **promoted**.
>
> If `refined` or `promoted`: produce updated `evidence.snippet`, `explanation`, `fix_sketch`, and (for promoted) a new `severity`. Explain in `notes` what the money-specific check revealed that wasn't in the original.
>
> If `rejected`: explain in `notes` what made it a false positive on closer money-specific examination.
>
> Output JSON only:
> ```json
> {
>   "id": "<finding id>",
>   "money_revalidation": {
>     "status": "...",
>     "notes": "..."
>   },
>   "updated_fields": { ... }   // only if status is refined or promoted
> }
> ```

The skill applies `updated_fields` to the finding after receiving the response. `updated_fields` may not change `id`, `primary_dimension`, or `secondary_dimensions` — only evidence/explanation/fix_sketch/severity.

---

## Specific checks the re-validation enforces

These are pulled from `dimensions/money-and-payments.md` and exercised against each finding in scope:

### M-RV-1 — Idempotency-key derivation

Stripe mutations must use an idempotency key derived from a **stable internal ID**, not `SecureRandom.uuid` or `Time.now`. A re-run of the same job/webhook with the same logical operation must produce the same key.

- Confirmed if the finding already cites this with the right derivation.
- Refined if the finding mentions "idempotency" but doesn't specify the derivation.
- Promoted if the original finding flagged a missing idempotency key as **medium**; promote to **high** (the rubric default for money mutations missing idempotency keys).

### M-RV-2 — Transaction boundary ordering

Multi-step money writes need this shape:

```ruby
Payout.transaction do
  payout = Payout.create!(...)                              # local DB first
  result = Stripe::Payout.create({...},                     # external second
                                  idempotency_key: "payout-#{payout.id}")
  payout.update!(stripe_payout_id: result.id, ...)          # local DB last
end
```

- Promote a finding to **blocker** if the original misses that the external call happens *before* the local record exists (orphan-state risk).
- Refine if the order is correct but transaction boundary is missing.

### M-RV-3 — Webhook event-ID dedup

Even with `Stripe::Webhook.construct_event` signature verification, a `processed_stripe_events` table with a unique constraint on `stripe_event_id` is required. Re-validation requires the dedup pattern in the cited file.

- Confirmed if the finding is already a blocker.
- Promoted from high → blocker if the cited handler runs irreversible side effects (charges, transfers, balance updates) on the dedup'd path.

### M-RV-4 — Money column types

Any money-shaped column (`amount`, `price`, `cost`, `fee`, `commission`, `payout`, `charge`, `balance`, `*_cents`) must be `Decimal` or `bigint`. `Float` is a **blocker**.

- Promoted to blocker on Float; rejected if the column turns out to be a non-money quantity (e.g. `weight`).

### M-RV-5 — Refund/reversal idempotency

Refund and reversal operations need their own idempotency key — they're high-risk on retry (already-refunded charges, partial refunds, etc.).

- Promote a finding from medium → high if it flags a missing refund idempotency key.

### M-RV-6 — Audit trail completeness

Each money-moving table needs columns: `external_reference_id`, `status` (with explicit state machine), `processed_at`, `error_code` / `error_message`, `idempotency_key`.

- Refine an existing finding to specify the missing column(s) explicitly.

---

## What re-validation deliberately does *not* do

- **Does not re-run agent fan-out.** Re-validation is a focused pass on existing findings, not a fresh audit.
- **Does not modify primary or secondary dimensions.** Tagging is set in synthesis (Step 4); re-validation respects it.
- **Does not introduce findings outside the proactive sweep paths.** If money issues exist in `lib/` or `engines/`, they should have been caught upstream.
- **Does not invoke external services.** All checks operate on source code + schema.

---

## Render

`output-template.md` already conditionally appends `_Re-validated: <status>_` to a finding entry when `money_revalidation.status` is set. v0.2 does not need template changes.

---

## Future work (v0.3+)

- **Re-validation for other high-stakes dimensions.** Currently money-only; expand to security-and-authz and authorization once their dimension files have similar specific-check checklists.
- **Cross-finding consistency.** If two findings disagree (one says payouts have idempotency keys, another says they don't), surface the contradiction.
- **External-API-shape verification.** When `tool_origin == "money-revalidation"` and the finding cites a Stripe API call, optionally hit the Stripe SDK's metadata to confirm the call shape exists in the project's Stripe gem version.
