# Dimension: Background jobs

Job stability is a leading indicator of overall stability — jobs are where retries, idempotency, and external state collide.

## What "good" looks like

- One job adapter, configured intentionally (Sidekiq / GoodJob / Cloudtasker / Resque).
- `ApplicationJob` defines `retry_on` and `discard_on` policies that derived jobs override only when needed.
- Every job that mutates external state is idempotent (re-running with same args produces same end state).
- Long-running jobs are split or chunked.
- Dead-letter / morgue queue exists for repeatedly-failing jobs.
- Job arguments are simple types (integers, strings) — not full ActiveRecord objects.
- Critical jobs have specs, including a "what if this runs twice" test.

## Checks

### Adapter

```bash
grep -E "gem ['\"](sidekiq|good_job|cloudtasker|resque|delayed_job|que|shoryuken)['\"]" Gemfile
grep "queue_adapter" config/environments/production.rb config/application.rb
```

If multiple adapters in Gemfile → confirm only one is active. If `:async` adapter in production → **Blocker** (in-process, lost on restart).

### `ApplicationJob` policies

```bash
cat app/jobs/application_job.rb
```

Required:
- `retry_on` for transient errors (network timeouts, deadlocks)
- `discard_on` for permanent errors (`ActiveJob::DeserializationError`, `ActiveRecord::RecordNotFound`)

If both are commented out / absent → **High** (every job inherits zero retry safety).

Common good baseline:

```ruby
class ApplicationJob < ActiveJob::Base
  retry_on ActiveRecord::Deadlocked, attempts: 3, wait: :exponentially_longer
  retry_on Net::OpenTimeout, attempts: 5, wait: :exponentially_longer
  discard_on ActiveJob::DeserializationError
  discard_on ActiveRecord::RecordNotFound
end
```

### Per-job overrides

For each file under `app/jobs/`:

```bash
grep -rn "retry_on\|discard_on\|sidekiq_options" app/jobs/ workers/ --include="*.rb"
```

Jobs that mutate external state (calls to Stripe, Twilio, GCS, etc.) need explicit retry policies. Audit each.

### Idempotency

For each job, read the body and answer: **if this runs twice, what happens?**

Patterns to surface:
- Job creates a record without a unique constraint → duplicates on retry.
- Job calls Stripe `create` without idempotency key → duplicate charge/transfer.
- Job sends an email without dedup → duplicate email.
- Job updates a counter with `increment!` (non-idempotent) instead of `update!(value: x)` (idempotent).

Each non-idempotent external mutation → **High** (or **Blocker** if money is involved).

### Argument shape

```bash
# Look for jobs that take whole AR objects as arguments
grep -rnE "perform_later\(.*<\.|perform_later\(\w+\.\w+\)" app/ --include="*.rb"
# Direct ActiveRecord pass works in dev with :async but breaks under serialization
grep -rn "GlobalID" app/jobs/ --include="*.rb"
```

If jobs receive whole records, GlobalID will reload them at job-run time — when the record is deleted or stale, the job fails. Prefer passing IDs and re-fetching.

### Long-running jobs

```bash
# Heuristic: jobs > 100 LOC
find app/jobs -name "*.rb" -exec wc -l {} + 2>/dev/null | sort -rn | head -10
```

Long jobs should be split. Look for loops over large collections without chunking → **Medium** (memory growth, retry replays whole loop).

### Dead-letter / morgue

Per adapter:
- **Sidekiq** — has built-in retry + morgue, but does anyone monitor the morgue queue?
- **Cloudtasker / Cloud Tasks** — failed tasks → no automatic morgue, need explicit alerting.
- **GoodJob** — has `cron_set`, retried jobs visible via web UI.

Read deploy/runtime configs. If failed jobs have no path to human attention → **High**.

### Job specs

```bash
ls spec/jobs/ 2>/dev/null
find app/jobs -name "*.rb" | wc -l
find spec/jobs -name "*_spec.rb" | wc -l
```

Per-job spec coverage. Jobs that touch money / external state without a spec → **High**. Jobs that don't have an "if this runs twice" spec → **Medium**.

### Job naming

Jobs named after side effects (`SendWelcomeEmailJob`) are clearer than jobs named after entities (`UserJob`). Concentrations of vague names → **Low**.

### Cron / scheduled jobs

```bash
# whenever / sidekiq-cron / good_job cron / cloudtasker cron
grep -E "gem ['\"](whenever|sidekiq-cron|sidekiq-scheduler)['\"]" Gemfile
test -f config/schedule.rb && wc -l config/schedule.rb
ls config/initializers/sidekiq*.rb 2>/dev/null
# Cloud Scheduler (Cloud Run flavor)
ls .github/workflows/ | grep -i schedul
grep -rn "cron" config/ 2>/dev/null
```

Scheduled jobs need:
- Lock to prevent double-runs across instances (advisory lock or distributed lock).
- Idempotency at the cron level (running an hourly job twice in one hour shouldn't break anything).

### Adapter-specific checks

#### Sidekiq

```bash
test -f config/sidekiq.yml && cat config/sidekiq.yml
grep "Sidekiq.configure" config/initializers/sidekiq*.rb 2>/dev/null
```

- Concurrency setting matches DB pool?
- Queues defined with priorities?
- Web UI mounted with auth (not public!)?

#### Cloudtasker / Cloud Tasks

```bash
grep -rn "Cloudtasker" config/initializers/ app/ --include="*.rb" | head
test -f config/initializers/cloudtasker.rb && cat config/initializers/cloudtasker.rb
```

- Queue name and location explicit per env?
- Auth token configured (HMAC on callback)?
- Max retries set on queue (Cloud Tasks queue config, not in code)?

#### GoodJob

```bash
grep "good_job" config/database.yml config/initializers/good_job*.rb 2>/dev/null
```

- Execution mode (`:async`, `:external`)?
- Cleanup of old `good_jobs` rows configured?

## Cross-cuts

- **`money-and-payments`** — payment jobs without retry/idempotency cross both.
- **`reliability`** — retries, dead-letter, locks are reliability primary if framing is "what happens when external services hiccup."
- **`observability`** — job runtime visibility, failed-job alerting cross with observability.
- **`security-and-authz`** — Sidekiq Web UI mounted without auth is security primary; jobs secondary.
- **`data-integrity`** — jobs that mutate state without transactions cross with integrity.

## Severity calibration

| Pattern | Default tier |
|---|---|
| `:async` adapter in production | Blocker |
| `ApplicationJob` with no `retry_on`/`discard_on` | High |
| Job with external mutation + no idempotency key | High (Blocker if money) |
| Job mutates external state + no spec | High |
| No path for failed jobs to human attention | High |
| Sidekiq Web UI exposed without auth | Blocker |
| Job receives full AR object instead of ID | Medium |
| Cron job without lock (multi-instance double-run risk) | High |
| Long-running job without chunking | Medium |
| Job count > spec count by >50% | High |
| Vague job names | Low |

## Reference points

- **[`sidekiq/sidekiq`](https://github.com/sidekiq/sidekiq)** — README + wiki are the canonical reference for retry/discard and worker patterns.
- **[`bensheldon/good_job`](https://github.com/bensheldon/good_job)** — Postgres-backed; their own test suite is a model for "what if this runs twice" specs.
- **[`keypup-io/cloudtasker`](https://github.com/keypup-io/cloudtasker)** — for projects on Google Cloud Tasks; README documents callback HMAC and worker conventions.
- **[Stripe's own architecture posts](https://stripe.com/blog/idempotency)** — idempotency at the API and job level.
- **[`Shopify/maintenance_tasks`](https://github.com/Shopify/maintenance_tasks)** — long-running, resumable jobs with progress tracking.

_Sidekiq's Pro/Enterprise features are documented separately; check which tier you're on._
