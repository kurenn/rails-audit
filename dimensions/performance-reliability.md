# Dimensions: Performance + Reliability

Performance = speed under expected load. Reliability = behavior under adverse conditions (timeouts, partial failures, retries).

## What "good" looks like

### Performance
- No N+1 in hot endpoints; `bullet` enabled in dev.
- Every FK has an index.
- Queries with `LIKE '%foo%'` either justified or backed by trigram/GIN.
- Caching strategy explicit (Russian-doll, fragment, low-level via `Rails.cache.fetch`).
- Long-running operations in background jobs, not requests.

### Reliability
- Every external HTTP call has a timeout — both connection and read.
- Outbound calls are wrapped in retries with exponential backoff (or a circuit breaker like `stoplight`).
- Web pool ≠ worker pool (separate connection pools or process types).
- Idempotency on every operation that could be retried (job, webhook, client-retry).
- Graceful degradation when external services are down (queue + retry, return cached, return 503 with retry-after).

## Performance checks

### N+1

```bash
# bullet gem
grep -E "gem ['\"]bullet['\"]" Gemfile
# .includes / .preload / .eager_load presence
grep -rnE "\.(includes|preload|eager_load)\(" app/controllers/ app/services/ --include="*.rb" | wc -l
# Direct AR queries in serializers/views (already in code-health.md, but cross-list)
grep -rnE "\.where\(|\.find_by|\.count\b" app/views/ app/serializers/ 2>/dev/null
```

If `bullet` is absent → **Medium** (turn it on in dev/test, lots of free signal).

### Index hygiene

Read `db/schema.rb` and grep for FK columns without indexes:

```bash
ruby -e '
schema = File.read("db/schema.rb") rescue nil
exit unless schema
schema.scan(/create_table[^"]*"(\w+)"[^d]*do \|t\|(.*?)end/m).each do |table, body|
  fk_cols = body.scan(/t\.(?:references|belongs_to|integer|bigint)\s+:(\w+)/).flatten
  fk_cols.select! { |c| c.end_with?("_id") }
  body_indexes = body.scan(/(?:t\.index|index)\s+\[?:?["]?(\w+)/).flatten
  unindexed = fk_cols - body_indexes
  unindexed.each { |c| puts "#{table}.#{c}" }
end
' 2>/dev/null
```

Each unindexed FK → **Medium**. On a large/hot table → **High**.

Polymorphic associations need a `[type, id]` composite index — flag absence as **High** (cross-listed in code-health).

### Slow query patterns

```bash
# LIKE with leading wildcard
grep -rnE "LIKE.*%#\{|LIKE.*%\?" app/ --include="*.rb"
# OR conditions (PG hates these)
grep -rnE "\.or\(" app/ --include="*.rb" | head
# unbounded list endpoints
grep -rnE "\.all\b|\.find_each\b" app/controllers/ --include="*.rb"
```

`LIKE '%term%'` without trigram index → **Medium**. Unbounded `.all` rendering → **High** (use pagination — `pagy`, `kaminari`).

### Caching

```bash
# Cache store config
grep -E "cache_store" config/environments/production.rb
# Usage
grep -rn "Rails.cache\|fragment_cache\|cache_key" app/ --include="*.rb" | wc -l
```

`memory_store` in production → **High** (per-process, breaks across instances). No caching at all on a high-traffic app → **Medium**.

### Asset pipeline

```bash
grep -E "(sprockets|propshaft|importmap-rails|jsbundling-rails|cssbundling-rails)" Gemfile
# CDN
grep "asset_host" config/environments/production.rb
```

No `asset_host` for an HTML-rendering app → **Medium** (assets served from app dynos = wasted compute).

## Reliability checks

### Timeouts on external calls

```bash
# Faraday default has no timeout; flag explicit timeout setup
grep -rnE "Faraday|HTTParty|Net::HTTP|RestClient" app/services/ app/jobs/ --include="*.rb" | head
grep -rn "open_timeout\|read_timeout\|timeout:" app/services/ app/jobs/ --include="*.rb"
# Stripe SDK has its own timeout
grep -rn "Stripe.api_request_timeout\|max_network_retries" config/initializers/ 2>/dev/null
```

External call without explicit timeout → **High**. Without retry/backoff → **Medium**.

### Circuit breakers

```bash
grep -E "gem ['\"](stoplight|circuit_breaker|semian)['\"]" Gemfile
```

Multi-tenant or high-stakes external integrations without a circuit breaker → **Medium** (wishlist for >100k req/day apps; not required smaller).

### Connection pools

```bash
cat config/database.yml | grep -A 2 production
# Worker process count expectation
grep -E "workers|threads|preload_app" config/puma.rb
```

Expected web concurrency = `workers * threads` per instance. DB pool must ≥ threads. Cloud Run / Heroku autoscaling can multiply this — note required Postgres `max_connections` headroom.

If the project uses Sidekiq / Cloudtasker / GoodJob, worker processes consume their own pool. Recommend connection pool sized as `RAILS_MAX_THREADS + sidekiq_concurrency` minimum.

### Idempotency at request level

For mutating endpoints likely to be retried by clients (mobile apps, especially):

```bash
grep -rn "idempotency_key\|Idempotency-Key\|x-idempotency" app/controllers/ --include="*.rb"
```

API for mobile clients without server-side idempotency on POST/PUT → **Medium** (mobile retries are a fact; without idempotency, double-creates happen).

### Graceful degradation

For each external dependency identified (Stripe, Twilio, GCS, identity provider), trace:
- What happens when it's down? Does the request hang? 500? Queue + retry?

This is a manual read, not a grep. For each, write 1 line in the report: "Stripe down → request 500s with full backtrace" or "GCS down → upload returns 503, no retry".

### Process resilience

```bash
# Sigterm handling for graceful shutdown
grep -rn "trap\|Signal\.trap" config/ app/ --include="*.rb"
# Puma workers boot timeout
grep -E "worker_timeout|worker_boot_timeout" config/puma.rb
```

## Severity calibration

| Pattern | Default tier |
|---|---|
| Hot endpoint with confirmed N+1 | High |
| External call without timeout | High |
| `cache_store: :memory_store` in production | High |
| Unindexed FK on large/hot table | High |
| Polymorphic association without composite index | High |
| Unbounded list endpoint (`.all` rendered) | High |
| `bullet` not in Gemfile (dev) | Medium |
| External call without retry/backoff | Medium |
| Unindexed FK on small table | Medium |
| `LIKE '%term%'` without trigram | Medium |
| No circuit breaker on a high-stakes integration | Medium |
| No CDN/asset_host | Medium |
| Pool sized below `workers * threads` | High |
| No graceful degradation path documented for critical external deps | Medium |
