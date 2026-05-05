# Rails Stability Audit — influapp-api

**Date:** 2026-05-04
**Audited commit:** `fc8354b` on `main`
**Mode:** standard
**Stack detected:** Rails 7.0.4 / Ruby 3.1.2 / cloud_run / cloudtasker / custom_jwt_identity_platform / rspec

> This is the markdown render of [`sample-report.json`](sample-report.json). The skill renders this file from the JSON via the rules in [`output-template.md`](../output-template.md). Any change you'd want here should land in the JSON or the template — not by editing this file directly.

---

## Executive summary

**Overall risk: 4/10** — Domain shape and spec foundations are solid (random order, WebMock, transactional fixtures, mocked Identity Platform). What sits below the line is the production hardening: Rails+Ruby are both off support, force_ssl/require_master_key are disabled, there's no health endpoint, the API token compare is timing-vulnerable, and money-moving paths lack idempotency keys. Most of these are 1–3 line fixes — sequenced so each phase unblocks the next.

**Top 3 blockers**:
1. Ruby and Rails are both EOL — `.ruby-version:1`
2. force_ssl disabled in production — `config/environments/production.rb:51`
3. Hardcoded callback secret in production deploy workflow — `.github/workflows/deploy-to-cloud-run.yml:233`

**Recommended next action:** Land Phase 1 (deploy safety) within 1–2 days before any further feature work.

---

## Self-check

The skill's calibration check raised 3 warnings:

- Severity inflation: 48% of findings tagged 'high' (threshold: 40%)
- Blocker over-use: 32% of findings tagged 'blocker' (threshold: 25%)
- Note: in v0.2 these are warn-only. Consider whether some blockers should demote to high after the fact.

**Unverified blockers** (1): the cited line numbers could not be confirmed by re-reading the file. Treat with skepticism: `f-a1b2c3d4e5f60008`

---

## Per-dimension scorecards

| # | Dimension | Score | What would make it a 10 |
|---|---|---|---|
| 1 | Foundation | 3/10 | Bump Ruby to 3.3+ and Rails to 7.1/7.2 (both currently EOL) |
| 2 | Domain Shape | 7/10 | Version /api/; move background_jobs/* off the unauthenticated tier |
| 3 | Spec Stability | 6/10 | Freeze time-sensitive specs; switch CI image from development to testing target |
| 4 | Deploy & CI | 3/10 | force_ssl, /healthz, require_master_key, externalize callback secret, documented rollback |
| 5 | Security & AuthN | 4/10 | secure_compare for token; rack-attack on auth endpoints; verify Firebase JWT signature |
| 6 | Authorization | 5/10 | Adopt Pundit/CanCan; add policy specs across Brand/Influencer split |
| 7 | Money & Payments | 3/10 | Stripe idempotency keys; webhook event-ID dedup; replace update_column(:stripe_customer_id) |
| 8 | Risk Hotspots | 5/10 | Refactor Campaign (211 LOC) and User (137 LOC) toward extracted services |
| 9 | Code Smells | 5/10 | Eliminate update_column on security/money columns; consistent service interface |
| 10 | Test Coverage | 4/10 | Backfill app/services/payments/* and 15 of 16 form objects |
| 11 | Performance | 6/10 | Add bullet; add timeouts on every external call (currently 0 grep hits) |
| 12 | Reliability | 3/10 | Enable retry_on/discard_on on ApplicationJob; idempotency on payouts; HTTP timeouts |
| 13 | Observability | 4/10 | Add error tracker (Sentry/Honeybadger); APM; PII out of custom log lines |
| 14 | Background Jobs | 4/10 | Retry policy on ApplicationJob; specs for evidence_ingest_job (131 LOC, untested) |
| 15 | Data Integrity | 6/10 | DB-level FK on remaining belongs_to; reconcile validates_uniqueness vs unique indexes |
| 16 | Data Governance | 4/10 | AR encryption on PII columns; audit log on money/auth models; hard-delete path for users |
| 17 | Developer Experience | 6/10 | CI runs single-job ~14min in development Docker target (should use testing target) |
| 18 | Cost & Scaling | 6/10 | Cloud Run max-instances cap; cleanup job for ActiveStorage variants |

Score guide: 0–3 unsafe / 4–6 risky / 7–8 acceptable / 9–10 strong.

---

## Punch list

### Blocker

#### B1. force_ssl disabled in production — Deploy & CI · Security & AuthN
**Location:** `config/environments/production.rb:51`
**Evidence:**
```
# config.force_ssl = true
```
**Why this is a blocker:** App accepts plain HTTP if any path bypasses Cloud Run's TLS terminator. JWT/auth headers and Stripe customer IDs travel unencrypted in those cases.
**Fix sketch:** Uncomment. Cloud Run honors X-Forwarded-Proto by default. _(≈5m)_

#### B2. require_master_key disabled — Deploy & CI
**Location:** `config/environments/production.rb:23`
**Evidence:**
```
# config.require_master_key = true
```
**Why this is a blocker:** App boots without a master key and only fails when the first credential read happens — minutes after a deploy, by which point the bad revision may have served traffic.
**Fix sketch:** Uncomment. Verify CI env injection has the key set. _(≈5m)_

#### B3. No health endpoint — Deploy & CI
**Location:** `config/routes.rb`
**Evidence:**
```
(no match for health|healthz|readyz|liveness)
```
**Why this is a blocker:** Cloud Run can't probe readiness or gate rollouts; bad revisions receive traffic before they're ready.
**Fix sketch:** get '/healthz' => 'health#show'; controller does ActiveRecord::Base.connection.execute('SELECT 1') and renders 200. _(≈1h)_

#### B4. Hardcoded callback secret in production deploy workflow — Deploy & CI · Data Governance
**Location:** `.github/workflows/deploy-to-cloud-run.yml:233`
**Evidence:**
```
COMPRESS_VIDEO_CALLBACK_SECRET=1234567890abcdef
```
**Why this is a blocker:** Either this exact mock value is what production runs with (callbacks are unauthenticated in practice), or it's overridden later but the workflow shows no override path.
**Fix sketch:** Move to GCP Secret Manager or GitHub repo secret; reference via ${{ secrets.COMPRESS_VIDEO_CALLBACK_SECRET }}. _(≈1h)_

#### B5. URI gem CVEs — Security & AuthN
**Location:** `Gemfile.lock`
**Evidence:**
```
uri (0.13.0)
```
**Why this is a blocker:** bundle-audit reports CVE-2025-27221 (Medium, userinfo leakage in URI#join/merge/+) and CVE-2025-61594 (Unknown, credential-leakage bypass over CVE-2025-27221).
**Fix sketch:** bundle update uri (target >= 1.0.4 or ~> 0.13.3). _(≈5m)_

#### B6. Production deploy points at staging bucket and DB — Deploy & CI · Data Governance
**Location:** `.github/workflows/ci-and-cd-production.yml`
**Evidence:**
```
# until stripe tests are done
```
**Why this is a blocker:** Production data persists into staging storage; any staging teardown is a prod data event.
**Fix sketch:** Restore prod env vars; complete Stripe tests on a gated branch. _(≈1h)_

#### B7. API token compared with == — Security & AuthN
**Location:** `app/controllers/api/authenticated_controller.rb:33`
**Evidence:**
```
raise StandardError, 'Unauthorized Request' unless application_token == extracted_token
```
**Why this is a blocker:** Timing-vulnerable on every authenticated request. Same pattern in unauthenticated_controller.rb:25.
**Fix sketch:** ActiveSupport::SecurityUtils.secure_compare(application_token, extracted_token.to_s) with length-equality guard. _(≈5m)_

#### B8. Ruby and Rails are both EOL — Foundation · Security & AuthN
**Location:** `.ruby-version:1`
**Evidence:**
```
ruby-3.1.2
```
**Why this is a blocker:** Brakeman flags both Ruby 3.1.2 (EOL 2025-03-31) and Rails 7.0.4 (EOL 2025-04-01). Neither receives security backports. Any Ruby/Rails CVE published after April 2025 is unpatched.
**Fix sketch:** Plan upgrade to Ruby 3.3+ and Rails 7.1 or 7.2 LTS. _(≈1w)_

### High

#### H1. Stripe webhook lacks event-ID dedup — Money & Payments · Security & AuthN
**Location:** `app/controllers/webhooks/stripe_controller.rb:11-46`
**Evidence:**
```
def handle_event(event)
  case event.type
  when 'payment_intent.succeeded'
```
Signature is verified via Stripe::Webhook.construct_event, but no Event.find_or_create_by(stripe_event_id: event.id) guard. Any new event type added without the inscription_charge? defensive check will double-fire.
**Fix:** processed_stripe_events table with unique index on stripe_event_id; insert-or-skip at top of handle_event. _(≈1d)_ · _Re-validated: confirmed_

#### H2. ApplicationJob retry/discard policies commented out — Background Jobs · Money & Payments · Reliability
**Location:** `app/jobs/application_job.rb:1-9`
**Evidence:**
```
class ApplicationJob < ActiveJob::Base
  # retry_on ActiveRecord::Deadlocked
  # discard_on ActiveJob::DeserializationError
end
```
Every job inherits zero retry safety. Payment jobs (PayInfluencer.perform!) silently lose work on transient Stripe errors.
**Fix:** Enable both. Add retry_on Net::OpenTimeout, attempts: 5, wait: :exponentially_longer. _(≈1h)_

#### H3. Zero timeouts on external HTTP calls — Reliability
**Location:** `app/services/`
**Evidence:**
```
(grep open_timeout|read_timeout|timeout: returns 0 hits in app/services/ and app/jobs/)
```
Faraday default is no timeout. A Stripe/Twilio/GCS slow response hangs the request thread until Puma kills the worker.
**Fix:** Configure default Faraday connection with open_timeout: 5, read_timeout: 30. Stripe SDK: Stripe.api_request_timeout = 30; Stripe.max_network_retries = 2. _(≈1d)_

#### H4. update_column(:stripe_customer_id) bypasses callbacks/audit — Money & Payments · Code Smells · Data Integrity
**Location:** `app/services/payments/stripe_service.rb:35`
**Evidence:**
```
@user.update_column(:stripe_customer_id, customer.id)
```
A payment-relevant column being written without validations or callbacks. If an audit gem (paper_trail/audited) is added later, this silently skips it. Re-validation also notes this column should carry an audit trail per M-RV-6 (audit-column completeness on money tables).
**Fix:** @user.update!(stripe_customer_id: customer.id); track customer-ID changes via paper_trail or an explicit StripeCustomerLink record. _(≈1h)_ · _Re-validated: refined_

#### H5. update_column(:is_blocked) on security-relevant column — Code Smells · Security & AuthN · Data Integrity
**Location:** `app/models/influencer.rb:43`
**Evidence:**
```
update_column(:is_blocked, true)
```
Bypasses callbacks/validations. If an audit log or notification callback is added later, this silently skips it.
**Fix:** Use update! so future audit/notification callbacks fire. If you genuinely need to skip callbacks, name the helper explicitly. _(≈5m)_

#### H6. No rate limiting — Security & AuthN
**Location:** `Gemfile`
**Evidence:**
```
(no rack-attack)
```
Login, password reset, signup, and unauthenticated background_jobs/* mass-mutation routes are unbounded.
**Fix:** Add rack-attack with throttles on auth/*, registration/*, and background_jobs/*. _(≈1d)_

#### H7. Stripe payouts/transfers missing idempotency keys — Money & Payments
**Location:** `app/services/payments/payout.rb:11-62`
**Evidence:**
```
Stripe::Transfer.create(...) / Stripe::Payout.create(...) without idempotency_key:
```
A retried payout job creates duplicate Stripe Transfers/Payouts. Re-validation re-read app/services/payments/payout.rb:11-62 and confirmed no idempotency_key: argument on the create calls.
**Fix:** Pass idempotency_key: "payout-#{payout.id}" (stable internal ID) on every Stripe::*.create mutation. Do NOT use SecureRandom or Time.now for the key. _(≈1d)_ · _Re-validated: promoted_

#### H8. Identity from unverified x-user-id header — Authorization · Security & AuthN
**Location:** `app/controllers/api/authenticated_controller.rb:55-64`
**Evidence:**
```
def current_user
  user = User.find_by(identity_platform_id: user_header)
```
The header value is trusted as-is — no Firebase JWT signature check.
**Fix:** Verify the Firebase JWT against the JWKS; extract sub from the verified payload. Discard x-user-id. _(≈1d)_

#### H9. Possible command injection in video versioner — Security & AuthN
**Location:** `workers/compress_video/lib/media/versioner.rb:211`
**Evidence:**
```
`#{["ffprobe", "-v", "quiet", "-print_format", "json", "-show_streams", "-show_format", src_path].map { |c| c.include?(" ") ? "\"#{c}\"" : c }.join(" ")}`
```
Backticks with a src_path whose origin chain ultimately includes user-uploaded filenames. The other two Open3.capture3 warnings on lines 142 and 159 are array-form (argv, no shell) — Brakeman false positives.
**Fix:** Replace backticks with Open3.capture3("ffprobe", "-v", "quiet", "-print_format", "json", "-show_streams", "-show_format", src_path). _(≈1h)_

#### H10. 8 API controllers without request specs — Test Coverage · Spec Stability
**Location:** `spec/requests/api/`
**Evidence:**
```
(missing for campaigns_controller, brand_controller, influencers_controller, background_jobs_controller, and 4 others)
```
Critical surface untested; one regression away from breakage.
**Fix:** One spec per controller — happy path + auth-failure + one validation case per action. _(≈1w)_

#### H11. No error tracker / APM in production — Observability
**Location:** `Gemfile`
**Evidence:**
```
(no sentry-*, honeybadger, bugsnag, rollbar, ddtrace, newrelic_rpm, skylight, appsignal, opentelemetry-*)
```
Errors only surface via Cloud Run logs. No transaction tracing on payment paths.
**Fix:** Add sentry-ruby + sentry-rails with before_send PII scrubbing as a minimum. _(≈1d)_

#### H12. No audit log on money/auth models — Data Governance · Money & Payments
**Location:** `Gemfile`
**Evidence:**
```
(no audited, paper_trail, or logidze)
```
No ledger of who changed is_blocked, subscribed_at, stripe_customer_id, or non_attended_count.
**Fix:** Add paper_trail on User, Influencer, Brand, Payout, Transaction. _(≈1d)_

### Medium

| ID | Finding | Location | Dimension(s) |
|---|---|---|---|
| M1 | Time-sensitive specs use Time.current without freeze | `spec/requests/api/user_controller_spec.rb` | Spec Stability |
| M2 | update_columns(attrs) mass-update on User | `app/models/user.rb:118` | Code Smells · Data Integrity |
| M3 | validates_uniqueness count > unique indexes in schema | `db/schema.rb` | Data Integrity |
| M4 | CI builds development Docker target instead of testing | `docker-compose.yml` | Developer Experience |

> 1 finding suppressed by `.audit-ignore.yml`: see Appendix D.

### Low

| Area | Count | Examples |
|---|---|---|
| RuboCop offenses (convention) | 142 | Style/StringLiterals, Layout/LineLength, Style/Documentation |
| TODO/FIXME comments | 23 | app/models/user.rb:4, app/services/payments/commission.rb:3 |

---

## Recommended fix sequence

### Phase 1 — Make deploys safe (1-2d)
- [ ] B1: force_ssl disabled in production (`config/environments/production.rb`)
- [ ] B2: require_master_key disabled (`config/environments/production.rb`)
- [ ] B3: No health endpoint (`config/routes.rb`)
- [ ] B4: Hardcoded callback secret in production deploy workflow (`.github/workflows/deploy-to-cloud-run.yml`)
- [ ] B5: URI gem CVEs (`Gemfile.lock`)
- [ ] B6: Production deploy points at staging bucket and DB (`.github/workflows/ci-and-cd-production.yml`)

### Phase 2 — Close auth & money holes (3-5d)
- [ ] B7: API token compared with == (`app/controllers/api/authenticated_controller.rb`)
- [ ] H1: Stripe webhook lacks event-ID dedup (`app/controllers/webhooks/stripe_controller.rb`)
- [ ] H2: ApplicationJob retry/discard policies commented out (`app/jobs/application_job.rb`)
- [ ] H3: Zero timeouts on external HTTP calls (`app/services/`)
- [ ] H4: update_column(:stripe_customer_id) bypasses callbacks/audit (`app/services/payments/stripe_service.rb`)
- [ ] H5: update_column(:is_blocked) on security-relevant column (`app/models/influencer.rb`)
- [ ] H6: No rate limiting (`Gemfile`)
- [ ] H7: Stripe payouts/transfers missing idempotency keys (`app/services/payments/payout.rb`)
- [ ] H8: Identity from unverified x-user-id header (`app/controllers/api/authenticated_controller.rb`)
- [ ] H9: Possible command injection in video versioner (`workers/compress_video/lib/media/versioner.rb`)

### Phase 3 — Backfill specs on critical paths (1-2w)
- [ ] H10: 8 API controllers without request specs (`spec/requests/api/`)
- [ ] M1: Time-sensitive specs use Time.current without freeze (`spec/requests/api/user_controller_spec.rb`)

### Phase 4 — Foundation upgrade + observability (1-3w)
- [ ] B8: Ruby and Rails are both EOL (`.ruby-version`)
- [ ] H11: No error tracker / APM in production (`Gemfile`)
- [ ] H12: No audit log on money/auth models (`Gemfile`)

### Phase 5 — Smell cleanup (2w)
- [ ] M2: update_columns(attrs) mass-update on User (`app/models/user.rb`)
- [ ] M3: validates_uniqueness count > unique indexes in schema (`db/schema.rb`)
- [ ] M4: CI builds development Docker target instead of testing (`docker-compose.yml`)

**Rationale:**
- _Phase 1_: Team needs safe rollback before any other change.
- _Phase 2_: Closes immediate exploit/loss surface.
- _Phase 3_: Lets future changes not regress.
- _Phase 4_: Strategic foundation upgrade that should not be rushed (Rails major can take a sprint).
- _Phase 5_: Sustainable hygiene work that runs as background without blocking releases.

---

## Appendix A — Tooling

| Tool | Status | Version | Notes |
|---|---|---|---|
| brakeman | ran | 7.0.x | 5 finding(s) |
| bundle-audit | ran | 0.9.2 | 3 finding(s) |
| reek | ran | 6.5.x |  |
| rails_best_practices | ran | 1.23.4 |  |
| rubocop | missing | — | in Gemfile but bundle install failed locally on nio4r native compilation; CI image runs it cleanly |
| simplecov | missing | — | in Gemfile; no coverage/.last_run.json present at audit time |
| rubycritic | missing | — | added to Gemfile by Step 0; pending bundle install resolution |
| flog | missing | — | added to Gemfile by Step 0; pending bundle install resolution |
| flay | missing | — | added to Gemfile by Step 0; pending bundle install resolution |
| bullet | missing | — | added to Gemfile by Step 0; pending bundle install resolution |
| mutant | skipped | — | deep mode only |

## Appendix B — Coverage map

| Path | Files | With spec | Line cov | Branch cov | Risk | Notes |
|---|---|---|---|---|---|---|
| `app/services/payments/` | 8 | 2 | — | — | critical | payout.rb (172 LOC), commission.rb (122 LOC) untested |
| `app/controllers/webhooks/` | 2 | 1 | — | — | critical | stripe_controller has request spec; replay/dedup not exercised |
| `app/services/identity_platform/` | 5 | 1 | — | — | critical | base_api_request.rb (89 LOC) untested |
| `app/forms/` | 16 | 1 | — | — | critical | only collaboration_request_form |
| `app/services/storage/` | 3 | 0 | — | — | high | multipart_upload_service.rb (142 LOC) untested |
| `app/jobs/` | 7 | 4 | — | — | high | evidence_ingest_job (131 LOC), 2 others untested |
| `app/controllers/api/` | 24 | 16 | — | — | high | 8 controllers missing request specs |
| `app/models/` | 42 | 22 | — | — | medium | half coverage; concentrated in highest-traffic models |

## Appendix C — Top risk hotspots by file size

| File | LOC | Layer | Has spec? |
|---|---|---|---|
| `app/models/campaign.rb` | 211 | model | partial |
| `app/controllers/api/user_controller.rb` | 196 | controller | yes |
| `app/controllers/api/payments/stripes_controller.rb` | 196 | controller | yes |
| `app/services/payments/payout.rb` | 172 | service | no |
| `app/models/evidence.rb` | 170 | model | yes |
| `app/controllers/api/registrations_controller.rb` | 161 | controller | yes |
| `app/controllers/callbacks/compress_video_controller.rb` | 156 | controller | partial |
| `app/models/campaign_location.rb` | 155 | model | yes |
| `app/services/storage/multipart_upload_service.rb` | 142 | service | no |
| `app/jobs/evidence_ingest_job.rb` | 131 | job | no |
| `app/services/payments/commission.rb` | 122 | service | no |
| `app/services/storage/signs_service.rb` | 119 | service | no |

## Appendix D — Ignored findings

| ID | Reason | Acknowledged by | Expires |
|---|---|---|---|
| `f-c1b2c3d4e5f60002` | Bullet gem will be added in the Q3 N+1 audit sprint (LIN-1234). Current performance is acceptable per Datadog dashboards. | abkuri88@gmail.com | 2026-09-30 |

> **Stale-ignore warning** (1): `f-deadbeefdeadbeef` does not match any current finding. Remove from `.audit-ignore.yml`.

## Appendix E — Cost recap

|        | Input tokens | Output tokens |
|---|---|---|
| Estimated | 52000 | 14000 |
| Actual    | —    | — |

---

*Generated by `/rails-audit` v0.2.0-alpha.1 — standard mode*
