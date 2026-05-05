# Rails Stability Audit — example-api

**Date:** 2026-05-04
**Audited commit:** `fc8354b` on `main`
**Mode:** standard
**Stack detected:** Rails 7.0.4 / Ruby 3.1.2 / Cloud Run / Cloudtasker / custom JWT (Identity Platform)

> This is an illustrative report based on a real audit, redacted. Use it to understand the format the skill produces — not as guidance for any specific real project.

---

## Executive summary

**Overall risk: 4/10** — Foundation is reasonable; deploy hygiene and money/auth idempotency are below the line for a payment-processing app.

**Top 3 blockers**:
1. `force_ssl = true` is commented out — `config/environments/production.rb:51`
2. No `/healthz` endpoint; Cloud Run rollouts cannot probe readiness — `config/routes.rb`
3. Production deploy points at *staging* bucket and *staging* DB until "stripe tests are done" — `.github/workflows/ci-and-cd-production.yml:29-30`

**Recommended next action:** Land Phase 1 (deploy safety) before any further feature work — these block safe rollback of any subsequent fix.

---

## Per-dimension scorecards

| # | Dimension | Score | What would make it a 10 |
|---|---|---|---|
| 1 | Foundation | 6/10 | Bump Ruby off 3.1 (EOL Mar 2025); run `bundle audit` clean |
| 2 | Domain shape | 8/10 | Version the `/api/` namespace; move `background_jobs` off the unauthenticated tier |
| 3 | Spec stability | 6/10 | Freeze time-sensitive specs; switch CI image to `testing` target |
| 4 | Deploy & CI | 3/10 | force_ssl, healthz, rollback path, externalize callback secret |
| 5 | Security | 4/10 | `secure_compare` for token; verify Firebase JWT signature server-side |
| 6 | Authorization | 5/10 | Pundit/CanCan; policy specs on Brand vs Influencer split |
| 7 | Money paths | 3/10 | Stripe idempotency keys; webhook event-ID dedup table |
| 8 | Risk hotspots | 5/10 | Refactor `Campaign` (211 LOC), `User` (137 LOC) toward extracted services |
| 9 | Code smells | 6/10 | Eliminate `update_column` on `is_blocked`; consistent service interface (`.perform!` everywhere) |
| 10 | Test coverage | 4/10 | Backfill `app/services/payments/*` and 15 of 16 form objects |
| 11 | Performance | 6/10 | Add `bullet`; `Brand#average_rating` needs `.includes(:reviews)` |
| 12 | Reliability | 4/10 | Enable `retry_on` on `ApplicationJob`; idempotency keys on payouts |
| 13 | Observability | 5/10 | Lograge + request_id in log_tags; PII in custom log lines |
| 14 | Background jobs | 4/10 | Retry policy on ApplicationJob; specs for evidence_ingest_job |
| 15 | Data integrity | 6/10 | Soft-delete query discipline on CollaborationRequest |
| 16 | Data governance | 5/10 | Hard-delete path for users with PII |
| 17 | Developer experience | 7/10 | CI is single-job + ~14 min; trim non-essential dev gems from CI image |
| 18 | Cost & scaling | 6/10 | Cloud Run `max-instances` cap; Stripe egress per request |

Score guide: 0–3 unsafe / 4–6 risky / 7–8 acceptable / 9–10 strong.

---

## Punch list

### Blocker

#### B1. `force_ssl` disabled in production — Deploy & CI
**Location:** `config/environments/production.rb:51`
**Evidence:**
```ruby
# config.force_ssl = true
```
**Why this is a blocker:** App accepts plain HTTP if any path bypasses Cloud Run's TLS terminator. JWT/auth headers and Stripe customer IDs travel unencrypted in those cases.
**Fix sketch:** Uncomment; verify `X-Forwarded-Proto` is honored (Cloud Run sets it correctly by default).

#### B2. No health endpoint — Deploy & CI
**Location:** `config/routes.rb` (entire file — no match for `health|healthz|readyz`)
**Why this is a blocker:** Cloud Run routes traffic to instances before they're ready, and bad revisions can be marked healthy on cold start. Rollouts cannot gate safely.
**Fix sketch:** Add `get "/healthz" => "health#show"`; controller does `ActiveRecord::Base.connection.execute("SELECT 1")` and renders 200.

#### B3. Production deploy uses staging bucket + staging DB — Deploy & CI
**Location:** `.github/workflows/ci-and-cd-production.yml:29-30`
**Evidence:** Comment `# until stripe tests are done` next to env vars pointing prod to staging resources.
**Why this is a blocker:** Production data persists to staging storage; any staging environment teardown deletes prod data.
**Fix sketch:** Restore prod env vars; complete the Stripe tests in a separate gated branch.

#### B4. Stripe webhook handler has no event-ID dedup — Money paths
**Location:** `app/controllers/webhooks/stripe_controller.rb:11-46`
**Evidence:**
```ruby
def handle_event(event)
  case event.type
  when 'payment_intent.succeeded'
    # ... no `Event.find_or_create_by(stripe_event_id: event.id)` guard
```
**Why this is a blocker:** Stripe retries on its own (network blips, our 5xx). Without event-ID dedup, the next event-type added without the inscription guard double-fires.
**Fix sketch:** `processed_stripe_events` table with unique index on `stripe_event_id`; insert-or-fail at the top of `handle_event`.

#### B5. Token comparison via `==` — Security
**Location:** `app/controllers/api/authenticated_controller.rb:33`
**Evidence:**
```ruby
raise StandardError, 'Unauthorized Request' unless application_token == extracted_token
```
**Why this is a blocker:** Timing attack on every authenticated request. Any leaked-API-token analysis becomes practical against a network attacker.
**Fix sketch:** `ActiveSupport::SecurityUtils.secure_compare(application_token, extracted_token.to_s)` with length-equality guard.

### High

#### H1. `ApplicationJob` retry/discard commented out — Background jobs
**Location:** `app/jobs/application_job.rb:1-9`
**Evidence:**
```ruby
class ApplicationJob < ActiveJob::Base
  # retry_on ActiveRecord::Deadlocked
  # discard_on ActiveJob::DeserializationError
end
```
**Why high:** Every job inherits zero retry safety. `ChangeToReviewedJob` calls `PayInfluencer.perform!`; one Stripe blip silently drops a payment.
**Fix sketch:** Enable both. Add `retry_on Net::OpenTimeout, attempts: 5, wait: :exponentially_longer`.

#### H2. Stripe payouts without idempotency keys — Money paths
**Location:** `app/services/payments/payout.rb:11-62`
**Evidence:** No `idempotency_key:` argument on `Stripe::Transfer.create` / `Stripe::Payout.create` calls.
**Fix sketch:** Pass `idempotency_key: "payout-#{payout.id}"` (stable internal ID, not random/time).

#### H3. Identity from unverified `x-user-id` header — Authorization
**Location:** `app/controllers/api/authenticated_controller.rb:55-64`
**Evidence:**
```ruby
def current_user
  user = User.find_by(identity_platform_id: user_header)
```
**Fix sketch:** Verify Firebase JWT signature against the JWKS; extract `sub` from the verified payload.

#### H4. Eight API controllers without request specs — Test coverage
**Location:** `spec/requests/api/` — missing for `campaigns_controller`, `brand_controller`, `influencers_controller`, `background_jobs_controller`, etc.
**Fix sketch:** One file per controller, happy path + auth-failure + one validation case.

(...continues with H5–H12 for brevity-redacted...)

### Medium

(...M1–M14...)

### Low

| Area | Count | Examples |
|---|---|---|
| RuboCop offenses (convention) | 142 | `Style/StringLiterals`, `Layout/LineLength` |
| TODO/FIXME comments | 23 | `app/models/user.rb:88`, `app/services/payments/commission.rb:48` |
| Dead/commented-out code blocks | 7 | `app/controllers/webhooks/stripe_controller.rb:18` |

---

## Recommended fix sequence

### Phase 1 — Make deploys safe (1–2 days)
- [ ] B1: Enable `force_ssl`
- [ ] B2: Add `/healthz` endpoint and wire to Cloud Run probe
- [ ] B3: Restore prod env vars; un-pin from staging
- [ ] H: Externalize `COMPRESS_VIDEO_CALLBACK_SECRET`
- [ ] H: Document a one-command rollback (`gcloud run deploy --image <prior-sha>`)

### Phase 2 — Close auth & money holes (2–3 days)
- [ ] B4: `processed_stripe_events` table + dedup
- [ ] B5: `secure_compare` on API token
- [ ] H1: Enable `retry_on`/`discard_on` on `ApplicationJob`
- [ ] H2: Idempotency keys on all `Stripe::*.create` mutations
- [ ] H3: Verify Firebase JWT signature server-side

### Phase 3 — Backfill specs on critical paths (3–5 days)
- [ ] H4: Request specs for the 8 missing controllers
- [ ] Spec coverage on `app/services/payments/*` to ≥80%
- [ ] Spec coverage on the 15 untested form objects to ≥80%
- [ ] Freeze time-sensitive request specs

### Phase 4 — Smell + perf cleanup (ongoing)
- [ ] Replace `update_column(:is_blocked, ...)` with normal save + audit
- [ ] Add `bullet` to dev/test
- [ ] Standardize service interface (`.perform!` everywhere)
- [ ] Fix `Brand#average_rating` N+1
- [ ] Convert `*100` / `/100` math to a single `Money` helper

Rationale: Phase 1 first because the team needs to ship the rest of the fixes safely; Phase 2 closes the immediate security/money holes; Phase 3 lets future changes not regress; Phase 4 is sustainable hygiene work that can run as background.

---

## Trend (no prior report)

First run — no trend data.

---

## Appendix A — Tooling

| Tool | Status | Version | Notes |
|---|---|---|---|
| brakeman | ran | 6.2.1 | 4 warnings (1 high, 3 medium) |
| bundle-audit | ran | 0.9.1 | 0 CVE in Gemfile.lock |
| rubocop | ran | 1.62 | 142 offenses (140 convention, 2 warning) |
| reek | missing | — | install: `gem 'reek', group: :development` |
| rails_best_practices | missing | — | install: `gem 'rails_best_practices'` |
| rubycritic | missing | — | optional |
| simplecov | partial | 0.22 | line coverage only — no branch coverage configured |
| flog | missing | — | optional |

## Appendix B — Coverage map

| Path | Line cov | Branch cov | Risk weight | Notes |
|---|---|---|---|---|
| `app/services/payments/` | 22% | n/a | critical | `payout.rb` 0%, `commission.rb` 0% |
| `app/controllers/webhooks/` | 50% | n/a | critical | stripe_controller has request spec but no replay test |
| `app/services/identity_platform/` | 40% | n/a | critical | `base_api_request.rb` (89 LOC) untested |
| `app/forms/` | 6% | n/a | critical | only `collaboration_request_form` has a spec |
| `app/services/storage/` | 0% | n/a | high | `multipart_upload_service.rb` (142 LOC) untested |
| `app/jobs/` | 57% | n/a | high | 4 of 7 jobs have specs |
| `app/controllers/api/` | 67% | n/a | high | 8 controllers missing request specs |
| `app/models/` | 45% | n/a | medium | 22 of 42 models have specs |

## Appendix C — Top risk hotspots by file size

| File | LOC | Layer | Has spec? |
|---|---|---|---|
| `app/models/campaign.rb` | 211 | model | yes (request-level) |
| `app/controllers/api/user_controller.rb` | 196 | controller | yes |
| `app/controllers/api/payments/stripes_controller.rb` | 196 | controller | yes |
| `app/services/payments/payout.rb` | 172 | service | **no** |
| `app/models/evidence.rb` | 170 | model | yes |
| `app/controllers/api/registrations_controller.rb` | 161 | controller | yes |
| `app/controllers/callbacks/compress_video_controller.rb` | 156 | controller | partial |
| `app/models/campaign_location.rb` | 155 | model | yes |
| `app/services/storage/multipart_upload_service.rb` | 142 | service | **no** |
| `app/jobs/evidence_ingest_job.rb` | 131 | job | **no** |

---

*Generated by `/rails-audit` — v0.1.0 — standard mode*
