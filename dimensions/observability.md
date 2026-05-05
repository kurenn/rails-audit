# Dimension: Observability

You can't operate what you can't see. Observability defects compound — a bug exists for weeks before it's noticed, by which point the cause is buried.

## What "good" looks like

- Structured logs (JSON) with at minimum: timestamp, level, request_id, user_id, route.
- Error tracker wired (Sentry / Honeybadger / Bugsnag / Rollbar) with `before_send` scrubbing.
- APM in production (Datadog / NewRelic / Skylight / AppSignal) — at least basic transaction traces.
- Request IDs propagated to background jobs (so you can trace a request → job → outbound call).
- Health endpoint distinct from metrics endpoint.
- Job-level instrumentation: run time, retry count, last failure reason.
- Custom business metrics for the things that matter (signups/day, payouts/day, commission paid/day) — not just RED/USE.

## Checks

### Logging

```bash
# Format
grep -E "config.log_formatter\|lograge\|semantic_logger\|rails_semantic_logger" Gemfile config/environments/production.rb config/initializers/ 2>/dev/null
# Tags
grep "config.log_tags" config/environments/production.rb
# Level
grep "config.log_level" config/environments/production.rb
```

Default Rails logger in production → **Medium** (Lograge or Semantic Logger gives JSON output). No `request_id` in `log_tags` → **High** (logs unjoinable across services). Log level `:debug` in production → **High** (PII leak risk + log volume cost).

### Error tracking

```bash
grep -E "gem ['\"](sentry-ruby|sentry-rails|honeybadger|bugsnag|rollbar|airbrake)['\"]" Gemfile
ls config/initializers/sentry*.rb config/initializers/honeybadger*.rb 2>/dev/null
```

No error tracker → **High** for a production app. Error tracker without scrubbing config (no `before_send` removing PII) → **Medium**.

### APM / tracing

```bash
grep -E "gem ['\"](ddtrace|datadog|newrelic_rpm|skylight|appsignal)['\"]" Gemfile
grep -E "gem ['\"](opentelemetry-)" Gemfile
```

No APM on production → **Medium** for a small app, **High** for revenue-critical paths.

### Request ID propagation

```bash
# Are jobs passed the request_id?
grep -rn "request_id\|X-Request-Id\|RequestStore" app/jobs/ app/services/ app/controllers/ --include="*.rb" | head
```

If background jobs don't carry request_id, debugging "why did job X fire?" becomes archaeology.

### Health and metrics endpoints

(Cross-listed with deploy-ci.md.)

Health and metrics should be separate routes:
- `/healthz` — liveness, no auth, fast (no DB unless DB-up is part of "live").
- `/readyz` — readiness, includes DB ping, dependent service checks.
- `/metrics` — Prometheus, behind internal-only auth.

```bash
grep -nE "health|healthz|readyz|metrics" config/routes.rb
```

### Audit logging

For compliance-relevant apps, an internal audit log of who-did-what is its own concern:

```bash
grep -E "gem ['\"](audited|paper_trail|logidze)['\"]" Gemfile
```

Money/auth-mutating actions without an audit trail (paper_trail, audited gem, or hand-rolled) → **High** for compliance contexts, **Medium** otherwise.

### Job observability

```bash
# Sidekiq
ls config/initializers/sidekiq*.rb 2>/dev/null
# Cloudtasker exposes per-task logs in GCP Logging by default
# GoodJob has a UI
```

Per-job runtime, success rate, retry count visibility — what's the path to see it? If the answer is "ssh in and grep" → **High**.

### Custom metrics

```bash
grep -rn "StatsD\|Datadog::Statsd\|Yabeda\|increment_counter\|metric" app/ --include="*.rb" | head
```

No custom metrics → **Low** for small apps, **Medium** for >$10k/month revenue apps (you'll regret it the first time something dips).

### PII scrubbing in logs and error tracker

```bash
# filter_parameters
grep "filter_parameters" config/initializers/filter_parameter_logging.rb config/application.rb
```

Required at minimum: `:password`, `:token`, `:secret`, `:api_key`, `:private_key`, `:authorization`, `:credit_card`. App-specific: email, phone, ID numbers, anything regulated. Missing → **High**.

Error tracker scrubbing (`before_send`):

```bash
grep -rn "before_send\|filter_parameters\|sanitize" config/initializers/sentry*.rb 2>/dev/null
```

### Alerting

This is mostly external (PagerDuty / Opsgenie / Slack), but check:
- Is there documented "what alerts go where" in README or runbooks?
- Does the deploy workflow notify on failure?

```bash
grep -i "slack\|pagerduty\|opsgenie\|teams\|notify" .github/workflows/*.yml 2>/dev/null
```

## Severity calibration

| Pattern | Default tier |
|---|---|
| Production app with no error tracker | High |
| Log level `:debug` in production | High |
| No request_id in log_tags | High |
| `filter_parameters` missing password/token | High |
| No audit log on money/auth operations (compliance context) | High |
| No APM on revenue-critical paths | High |
| Default Rails logger (no JSON / Lograge) in prod | Medium |
| No request_id propagation to jobs | Medium |
| No separate `/readyz` from `/healthz` | Medium |
| No custom business metrics | Low–Medium (depends on app size) |
| No alert routing for deploy failures | Medium |
