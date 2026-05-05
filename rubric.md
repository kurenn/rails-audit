# Severity rubric

Use these definitions verbatim. Don't invent tiers. Don't inflate severity to seem useful — calibration is the point of the rubric.

## Blocker

A finding is a **blocker** if any of the following is true *today*:

- **Data loss** is possible (e.g. destructive migration without backup verification, soft-delete query that returns deleted rows in critical paths).
- **Money loss** is possible (e.g. non-idempotent webhook that double-credits, payout without idempotency key, missing FX rounding guard).
- **Account takeover** is possible (e.g. unverified Firebase header used as identity, JWT `alg: none` accepted, password reset token without expiry, IDOR on user-owned resources).
- **Deploy can fail silently** (e.g. no health endpoint and Cloud Run rollouts mark bad revisions healthy, hardcoded secret will deploy to prod).
- **Compliance is violated** (e.g. prod data persisted to staging bucket, PCI scope leak, PII unencrypted at rest where required).
- **App will not boot** under realistic prod conditions (e.g. missing env var with no defensive check, `require_master_key` disabled and credential chain broken).

Money/auth/webhook defects default to blocker unless you can demonstrate a guard that prevents exploitation today.

## High

A finding is **high** if it's not exploitable/breaking today, but is **one mistake away** from being a blocker:

- No retry/idempotency on a job that mutates external state (next deploy with a Stripe blip = lost payment).
- No rollback strategy (next bad deploy = manual `gcloud run deploy` to recover, MTTR measured in hours).
- Critical-path file with zero spec coverage (next refactor = silent regression).
- Token comparison via `==` instead of `secure_compare` (timing attacks are slow but compounding).
- DB pool not sized for runtime concurrency (next traffic spike = connection exhaustion).
- Webhook signature verified but no event-ID dedup (Stripe retry semantics will eventually fire twice).

## Medium

A finding is **medium** if it represents correctness or performance risk under load or growth, but not under current conditions:

- N+1 queries in non-hot endpoints.
- Fat controllers/models above thresholds (controller action >25 lines, model >300 lines).
- Soft-delete inconsistency where the affected paths are low-traffic.
- Generic `rescue StandardError` conflating 4xx and 5xx (operational pain, not user-facing breakage).
- Callback chains >5 deep on a model.
- Train wrecks (`a.b.c.d.e`) in code that's actively edited.
- Direct ActiveRecord calls from views/serializers without preloading.

## Low

Hygiene and maintainability — these are real, but they don't move the stability needle alone:

- Style/formatting inconsistency outside hot paths.
- Magic numbers in non-money code.
- Commented-out code.
- TODO/FIXME density without a tracking system.
- Missing `.editorconfig` or `bin/setup`.
- README staleness in non-onboarding sections.
- Duplication outside critical paths.

## Calibration examples (from real audits)

| Finding | Tier | Why |
|---|---|---|
| `force_ssl = true` commented out in `production.rb` | Blocker | Auth + payment data unencrypted edge-to-app today |
| `application_token == extracted_token` in API auth | Blocker | Timing attack on every authenticated request; auth is broken-in-principle |
| No `/healthz` endpoint on Cloud Run | Blocker | Bad revisions receive traffic; rollouts can't gate |
| Stripe webhook lacks event-ID dedup table | High | Signature verified, retries are rare, but inevitable |
| `ApplicationJob` retry/discard commented out | High | Next transient failure on a payment job = silent loss |
| `update_column(:is_blocked, true)` on `Influencer` | High | Bypasses callbacks; if an audit log is added later, this silently skips it |
| `Brand#average_rating` without `.includes(:reviews)` | Medium | N+1, but the endpoint is admin-only and low-traffic |
| Controller action 47 lines, mostly straight-line | Medium | Smell, not bug — refactor target |
| 8 API controllers without request specs | High | Critical surface untested; one regression away from breakage |
| 60% of services have <50% line coverage | Medium | Real, but synthesize per-path: payments at 22% is high; tutorials at 18% is medium |
| TODO comment density 2.4/file | Low | Hygiene; track it but don't lead with it |

## Anti-patterns to avoid in the rubric

- **Don't invent "Critical" or "Severe" tiers.** Four tiers is the contract.
- **Don't list everything as high.** A report where 60% of findings are "high" is uncalibrated and gets ignored.
- **Don't soften money/auth findings.** If a payment endpoint has no idempotency, that's blocker-tier even if it "hasn't happened yet" — the rubric is about exploitability, not history.
- **Don't promote style issues to medium.** RuboCop output goes in the appendix as a count, not in the punch list as findings.
