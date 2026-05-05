# Dimensions: Foundation + Deploy & CI

Audit the deploy pipeline, prod config, secrets handling, and bootability.

## What "good" looks like

- Ruby and Rails are on supported (non-EOL) versions.
- Dockerfile multi-stage with a pinned base image (digest, not floating tag), runs as non-root, has a health probe.
- Production config: `force_ssl = true`, `require_master_key = true`, eager_load on, sensible log level, structured logger.
- A `/healthz` (or equivalent) endpoint that does a lightweight DB ping and returns 200; wired into Cloud Run / Heroku / k8s probes.
- Secrets only via env or `credentials.yml.enc` — never in workflow files, never in tracked `.env`.
- Workflows pinned: GitHub Actions referenced by SHA (or at minimum `@vN.x.x`), not `@main`.
- Documented rollback path: image tagged with both git-SHA and a moving tag; a workflow or runbook to redeploy a prior digest.
- DB pool sized to match the runtime concurrency (web threads × instances vs Postgres `max_connections`).

## Checks

### Foundation

```bash
cat .ruby-version 2>/dev/null
grep "^ruby\|gem ['\"]rails['\"]" Gemfile
# Ruby EOL: https://endoflife.date/ruby — Ruby 3.0 EOL'd Mar 2024, 3.1 EOL'd Mar 2025
# Rails EOL: 6.1 EOL'd Oct 2024, 7.0 maintenance only
bundle outdated --strict 2>/dev/null | head -40
bundle exec bundle-audit check --update 2>/dev/null
```

Note: if Ruby ≤ 3.1 or Rails ≤ 7.0, treat as **High** (security patches no longer back-ported). Active CVE in `bundle-audit` → **Blocker**.

### Boot health

```bash
test -f config/boot.rb && grep -i "bootsnap" config/boot.rb
test -f config/application.rb && grep -i "config.load_defaults" config/application.rb
grep -E "config.eager_load\b" config/environments/production.rb
```

`config.eager_load = true` in production is mandatory — flag if commented or false.

### Production config — read in full

```bash
cat config/environments/production.rb
```

Mandatory checks (each is **Blocker** if missing/misconfigured for a production app):

| Setting | Expected | Common defect |
|---|---|---|
| `config.force_ssl` | `true` | Commented out |
| `config.require_master_key` | `true` (or env-managed) | Commented out → boots on missing key, fails at first credential read |
| `config.eager_load` | `true` | False = slow first request, autoload thread issues |
| `config.log_level` | `:info` or stricter | `:debug` leaks PII |
| `config.log_tags` | request_id at minimum | Missing → unjoinable logs |
| `config.assets.compile` | `false` | True = on-the-fly compilation in prod |
| `config.action_mailer.raise_delivery_errors` | `true` | False hides mail failures |
| `config.active_record.dump_schema_after_migration` | (any) | Verify intentional |

### Health endpoint

```bash
grep -nE "health|healthz|liveness|readiness" config/routes.rb
# If found, read the controller action — should DB-ping and respond fast
```

If absent → **Blocker** for any platform that probes (Cloud Run, k8s, Heroku for some buildpacks).

### Database config

```bash
cat config/database.yml
```

- Production pool: `pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>` is the default; flag if static low number with high concurrency.
- `prepared_statements: false` is needed behind PgBouncer transaction mode — flag if PgBouncer is in use without it.
- `sslmode: require` (or stricter) for managed Postgres — flag if missing.

### Secrets

```bash
# Tracked env files
git ls-files | grep -E "\.env(\..*)?$"
# Keys in workflows
grep -rE "(sk_live_|sk_test_|AIza|xox[baprs]-|ghp_|ghs_|gho_|github_pat_|AKIA[0-9A-Z]{16}|aws_access)" .github/ 2>/dev/null
# Hardcoded "secrets" in workflows that look fake but might deploy
grep -rE "(SECRET|TOKEN|KEY)=[a-zA-Z0-9_]{8,}" .github/workflows/ 2>/dev/null
test -f config/master.key && echo "master.key present locally"
test -f config/credentials.yml.enc && echo "credentials encrypted"
```

Any committed `.env` (other than `.env.example`) → **Blocker**. Hardcoded production-shaped secret in a workflow → **Blocker**.

### Dockerfile

```bash
cat Dockerfile 2>/dev/null
```

Checks:
- `FROM ruby:X.Y.Z-...@sha256:<digest>` — pinned by digest? If just `:tag`, **Medium**.
- Multi-stage? Final image should not contain `node_modules`/build tools/test gems.
- `USER` directive — running as non-root? **Medium** if root.
- `HEALTHCHECK` instruction — present? Less critical when platform does probing.
- `.dockerignore` covers `.git`, `tmp/`, `log/`, `coverage/`, `node_modules/`, `spec/`?

### CI/CD workflows

For each file under `.github/workflows/`:

```bash
ls .github/workflows/
# For each:
cat .github/workflows/<file>.yml
```

For each workflow, document:
- Trigger (`on:`)
- Required secrets (`secrets.*` references)
- Action pin discipline — `uses: actor/action@v3` (loose) vs `@<sha>` (strict). Loose pins on third-party actions = **Medium**, on first-party (`actions/*`) = acceptable.
- Concurrency control (`concurrency:` block) — parallel deploys to the same env without locks → **High**.

Specifically for the deploy workflow, verify:
- Image tag includes git-SHA (so rollback works).
- Both `:latest`/floating and `:<sha>`/immutable tags exist.
- A documented or scripted rollback path. If absent → **High**.
- No `if:` conditions that silently skip steps.
- Database migrations: are they run? Where? Is there a guard against destructive migrations?

### Migration safety

```bash
# Last 10 migrations
ls -t db/migrate/*.rb 2>/dev/null | head -10
```

For each, look for:
- `add_column ..., null: false` without `default:` on a non-empty table → **High** (lock + backfill).
- `remove_column` without an `up`/`down` block → **Medium** (irreversible).
- `add_index` without `algorithm: :concurrently` on a large table → **High**.
- Data backfill in a schema migration (use `data_migrate` or rake task instead) → **Medium**.

### Background job runtime config

```bash
cat config/cable.yml
cat config/storage.yml
# Job adapter
grep "config.active_job.queue_adapter" config/environments/production.rb config/application.rb
```

If `cable.yml` falls back to `redis://localhost:6379` without env-based config → **High** (silent in-memory fallback breaks across instances).

### Rollback drill (deep mode)

In deep mode, ask the user (don't actually trigger): "If a deploy went bad right now, what's the rollback command? Walk me through it." Capture answer in the report — undocumented rollback is a **High** finding.

## Severity calibration

| Pattern | Default tier |
|---|---|
| `force_ssl` disabled in production.rb | Blocker |
| `require_master_key` disabled | Blocker |
| No /healthz endpoint with platform probing | Blocker |
| Hardcoded prod-shape secret in workflow | Blocker |
| Active CVE in bundle-audit | Blocker |
| Tracked `.env` with real values | Blocker |
| Prod points at staging bucket/DB | Blocker |
| Ruby/Rails on EOL version | High |
| No documented rollback path | High |
| Dockerfile base image not SHA-pinned | Medium |
| Loose third-party action pin | Medium |
| Container running as root | Medium |
| No `.dockerignore` or sloppy coverage | Low |
