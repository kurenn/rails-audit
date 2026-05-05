# Dimensions: Developer experience + Cost & scaling

DX gates how fast the team can fix the other 16 dimensions. Cost & scaling surfaces problems that haven't bitten yet but will as growth continues.

## Developer experience

### What "good" looks like

- `bin/setup` works on a fresh checkout.
- README's "Getting started" is accurate.
- Local dev mirrors production reasonably (Docker compose, reset_db scripts).
- Test suite runs in <5 minutes for normal feedback, <15 minutes total.
- Clear contribution path: PR template, CI feedback within 10 minutes.
- Onboarding tested — at least one engineer joined in last 12 months and didn't write a how-it-actually-works doc separately.

### Checks

```bash
test -x bin/setup && cat bin/setup
test -f README.md && wc -l README.md
test -f CONTRIBUTING.md
test -f .github/PULL_REQUEST_TEMPLATE.md .github/pull_request_template.md 2>/dev/null
test -f docker-compose.yml -o -f docker-compose.yaml
test -f .devcontainer.json -o -f .devcontainer/devcontainer.json
```

### Test suite runtime

```bash
# CI duration over last 10 runs (manual via gh CLI)
gh run list --workflow ci.yml --limit 10 2>/dev/null
# Local
grep -E "rspec|minitest" Gemfile.lock | head
```

If CI > 15 minutes consistently → **Medium** (saps PR velocity).

### bin/setup actually works

In `--deep` mode, the audit can attempt this. Otherwise note absence. A `bin/setup` that hasn't been touched in 18 months and references a Postgres version 4 versions back → **High** (new engineers will spend a day fighting it).

### Documentation freshness

```bash
# When was README last edited
git log -1 --format="%ad" -- README.md
git log -1 --format="%ad" -- CLAUDE.md AGENTS.md 2>/dev/null
# When was the latest commit
git log -1 --format="%ad"
```

README untouched for >6 months in an actively-developed app → **Medium**. Worth verifying it still works.

### Decision log / ADRs

```bash
ls docs/adr/ docs/decisions/ ARCHITECTURE.md 2>/dev/null
```

For an app of any complexity, having no documented architectural decisions → **Low** (Medium if the app has had 3+ rewrites of the same subsystem).

### `.editorconfig` / formatting consistency

```bash
test -f .editorconfig
test -f .rubocop.yml
test -f .rubocop_todo.yml
```

Missing `.rubocop.yml` → **Low** (RuboCop runs anyway, but no project-specific config means defaults).

### Local vs prod parity

```bash
# Compare gemfile groups
grep -E "^group :" Gemfile
# Same DB engine in dev as prod?
grep -E "adapter:" config/database.yml
```

`sqlite3` in dev + `pg` in prod → **Medium** (drift; PG-specific features fail at deploy).

## Cost & scaling

### What "good" looks like

- Largest tables identified, growth rate known, periodic archiving plan.
- Hot endpoints identified, performance budget per endpoint.
- Storage costs predictable (image variants cleaned up, GCS lifecycle policies).
- Background job concurrency tuned (not over-provisioned).
- Cloud Run / Heroku scaling config matches traffic.

### Checks

### Table size estimation

If the audit can connect to a DB (deep mode), run:

```sql
-- Postgres
SELECT
  schemaname, relname,
  pg_size_pretty(pg_total_relation_size(relid)) AS total_size,
  n_live_tup AS rows
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(relid) DESC
LIMIT 20;
```

Otherwise note in the appendix: "Run this to capture top tables."

### Index bloat

If on Postgres:

```sql
SELECT relname, pg_size_pretty(pg_relation_size(indexrelid))
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC
LIMIT 20;
```

Indexes >50% the size of their table → **Medium** (consider REINDEX or drop unused).

### Hot endpoint cost

```bash
# Look for unbounded operations in controllers
grep -rnE "\.all\b|\.find_each\b" app/controllers/ --include="*.rb"
# Pagination presence
grep -E "gem ['\"](pagy|kaminari|will_paginate)['\"]" Gemfile
```

Hot endpoint with `.all.map { ... }` rendering → **High**. No pagination gem on a list-heavy app → **Medium**.

### Storage / blob cost

```bash
# ActiveStorage
grep -rn "has_one_attached\|has_many_attached" app/models/ --include="*.rb"
# Variants (each image variant doubles storage)
grep -rn "variant\|representation" app/ --include="*.rb"
# Lifecycle / cleanup
grep -rn "purge\|cleanup" app/jobs/ lib/tasks/ 2>/dev/null
```

ActiveStorage with many variants and no cleanup job → **Medium** (storage grows linearly with users).

### Job concurrency tuning

```bash
# Sidekiq
test -f config/sidekiq.yml && grep "concurrency" config/sidekiq.yml
# Cloudtasker — queue config in GCP
# GoodJob
grep -E "good_job_max_threads|max_threads" config/initializers/good_job*.rb 2>/dev/null
```

Default concurrency (10–25) on a small workload → **Low** (over-provisioned, costs more than needed). Default concurrency on a large workload → **Medium** (under-provisioned, queue depth grows).

### Scaling configuration

#### Cloud Run

```bash
grep -rn "min-instances\|max-instances\|cpu-boost\|concurrency" .github/workflows/ 2>/dev/null
```

`min-instances: 0` on a customer-facing service → **Medium** (cold starts).
`max-instances: <unbounded>` → **High** (single bug = unbounded cost).
No `concurrency` set → defaults to 80, sometimes too high for Rails (each instance can saturate DB pool).

#### Heroku

```bash
test -f Procfile && cat Procfile
test -f app.json && cat app.json
```

Default web dyno on a real-traffic app → **Medium**. No `release:` command for migrations → **Medium**.

### Asset / page weight

If HTML-rendering app:
- JS bundle size (importmap is ~free; jsbundling can balloon)
- CSS framework cost (Tailwind purge configured)

```bash
# Build artifact sizes
ls -la public/assets/ 2>/dev/null | head
```

### Egress costs

External API calls per request × requests/day = $/month. Stripe/Twilio especially. Unmonitored → **Low** (track but don't lead).

## Cross-cuts

- **`spec-stability`** — slow CI / flake retries are stability primary; DX secondary.
- **`deploy-and-ci`** — Dockerfile choices, CI mechanics, environment parity cross both.
- **`performance`** — table size growth and missing indexes are perf primary; cost secondary.
- **`reliability`** — Cloud Run scaling config is reliability primary; cost secondary.
- **`developer-experience`** — `bin/setup`, README freshness, ADRs are DX primary if onboarding-related.

## Severity calibration

| Pattern | Default tier |
|---|---|
| `bin/setup` broken or absent | High |
| `sqlite` dev + `pg` prod | Medium |
| README untouched >6 months on active app | Medium |
| CI runtime >15 min | Medium |
| `.all.map` on hot list endpoint | High |
| No pagination gem on list-heavy app | Medium |
| ActiveStorage variants without cleanup | Medium |
| Cloud Run `max-instances` unbounded | High |
| Cloud Run `min-instances: 0` on customer-facing | Medium |
| Sidekiq concurrency mismatch with workload | Medium |
| Index bloat >50% table size | Medium |
| No table-size monitoring/archiving plan on tables >1M rows | Medium |
| No PR template / CONTRIBUTING.md (team >3) | Low |
| No ADR / decision log | Low |

## Reference points

- **[`thoughtbot/suspenders`](https://github.com/thoughtbot/suspenders)** — Rails template with opinionated `bin/setup` that actually works on a fresh checkout.
- **[`gitlabhq/gitlabhq`](https://gitlab.com/gitlab-org/gitlab)** development docs — onboarding and dev-environment parity at scale.
- **[`discourse/discourse`](https://github.com/discourse/discourse)** — `bin/docker_dev_env` and developer docs are well-maintained.
- **[`evilmartians/anyway_config`](https://github.com/palkan/anyway_config)** — for env-config consistency between dev and prod.
- **[`pagy`](https://github.com/ddnexus/pagy)** — fastest Ruby paginator; for projects with cost-sensitive list endpoints.
- **[Cloud Run scaling docs](https://cloud.google.com/run/docs/about-instance-autoscaling)** — concurrency / min-instances / max-instances tradeoffs.

_Rails `bin/setup` patterns vary widely; consider it a smell if the repo's hasn't been touched in >12 months._
