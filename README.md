# rails-audit

A Claude Code skill that runs a structured stability audit of a Ruby on Rails codebase across 18 dimensions and produces a single severity-ranked markdown report with a recommended fix sequence.

## Why

Rails audits in practice tend to fall into two failure modes:

1. **A single tool's output dressed up as an audit.** A 400-warning `brakeman` dump is not a report — it's evidence. Engineers ignore it.
2. **An exhaustive checklist with no severity calibration.** Everything is "high" → nothing is high → no one acts.

This skill takes the opposite approach. It invokes the existing Rails tooling ecosystem (`brakeman`, `bundler-audit`, `rubocop`, `reek`, `rails_best_practices`, `rubycritic`, `simplecov`, `flog`/`flay`) plus codebase-aware grep and orchestrates 4 parallel subagents to **synthesize** findings into:

- An executive summary with a 0–10 risk score
- Per-dimension scorecards
- A punch list grouped by Blocker / High / Medium / Low — with verbatim evidence and fix sketches
- A recommended fix sequence ordered so each phase unblocks the next
- A trend table when re-run on the same repo

## What it covers

18 dimensions, organized into 4 clusters that run in parallel:

**Cluster A — Spec & Coverage**
1. Spec stability — flake patterns, factory cascade, CI mechanics
2. Test coverage — risk-weighted coverage, branch coverage, mutation testing on critical paths

**Cluster B — Deploy & CI**
3. Foundation — Ruby/Rails EOL, lock freshness, boot health
4. Deploy & CI — workflows, Dockerfile, prod config, secrets, health, rollback
5. Observability — structured logs, error tracking, APM, request IDs

**Cluster C — Code Health**
6. Domain shape — model/route inventory, layering
7. Risk hotspots — biggest files per layer, churn × complexity
8. Code smells & antipatterns — Rails-specific + generic Ruby smells, callback chains, train wrecks, fat controllers
9. Performance — N+1, indexes, cache, slow queries
10. Reliability — timeouts, circuit breakers, pool sizing, idempotency
11. Background jobs — adapter, retry/discard, idempotency, DLQ
12. Data integrity — FK constraints, validation duplication, soft-delete consistency
13. Developer experience — `bin/setup`, suite runtime, README accuracy
14. Cost & scaling — table sizes, missing indexes, hot-endpoint N+1

**Cluster D — Security & Money**
15. Security — OWASP Top 10 mapped to Rails (SSRF, deserialization, injection, headers, secrets in logs)
16. Authorization — IDOR, policy spec coverage, admin-action testing
17. Money paths — Decimal vs Float, Stripe idempotency, webhook event dedup, transaction boundaries
18. Data governance — PII inventory, encryption at rest, audit logs, retention

## Modes

| Mode | Time | What runs |
|---|---|---|
| `--quick` | ~3 min | Static tools + grep checks, single agent. PR-time use. |
| `--standard` (default) | ~10–15 min | Full audit, 4 parallel subagents. |
| `--deep` | ~30+ min | Standard + boots app, runs spec subset, mutation testing on critical paths. |

## Installation

### Recommended — via the kurenn marketplace

```bash
claude plugin marketplace add kurenn/marketplace   # one-time per user
claude plugin install rails-audit@kurenn           # one-time install
```

After install, restart your Claude Code session and `/rails-audit` appears in the slash menu.

### Local plugin dir (development)

```bash
git clone https://github.com/kurenn/rails-audit ~/workspace/rails-audit
claude --plugin-dir ~/workspace/rails-audit
```

### Manual install (legacy, pre-plugin layout)

If you're on a Claude Code version that predates plugin support, clone directly into the skills directory:

```bash
git clone https://github.com/kurenn/rails-audit.git ~/.claude/skills/rails-audit-legacy
# Then move skills/rails-audit/ contents to the right place, or update the path
```

> As of 0.3.0 the skill content lives under `skills/rails-audit/` (plugin layout). Older clones expecting it at the repo root won't find SKILL.md anymore.

## Updating

When a new version is released, pull it via the marketplace:

```bash
# Refresh the marketplace cache (picks up new versions from marketplace.json)
claude plugin marketplace update kurenn

# Update rails-audit to the latest released version
claude plugin update rails-audit
```

Restart your Claude Code session after updating so the slash menu picks up the new version.

Verify which version you're on:

```bash
claude plugin list | grep -A3 rails-audit@kurenn
```

To pin to an older version (e.g. for rollback while debugging a regression), clone the tag locally and load via `--plugin-dir`:

```bash
git clone https://github.com/kurenn/rails-audit --branch v0.3.0 ~/workspace/rails-audit-0.3.0
claude --plugin-dir ~/workspace/rails-audit-0.3.0
```

## Usage

In Claude Code, with a Rails project as your working directory:

```
/rails-audit
```

or

```
/rails-audit --quick
/rails-audit --deep
```

The skill writes the report to `tmp/rails-audit/report-YYYY-MM-DD.md` and replies with a ≤200-word summary linking to the file.

## Standalone tool inventory

```bash
bin/check-tools                  # human-readable table
bin/check-tools --json           # machine-readable (matches the report.json `tooling{}` block)
bin/check-tools --required-only  # exits 1 if any Tier-1 tool is missing — useful as a CI pre-flight
```

Run from any Rails project root. After plugin install, the script lives at `~/.claude/plugins/cache/kurenn/rails-audit/<version>/bin/check-tools` (or wherever Claude caches plugins on your system). Reads `tooling.md` for tier definitions, then detects each tool by Gemfile presence + binary in PATH.

## Project profile (optional)

Tell the skill about your stack so checks adapt. Create `.claude/rails-audit.yml` at repo root:

```yaml
deploy_target: cloud_run         # cloud_run | heroku | kamal | render | ecs | fly | other
job_adapter: cloudtasker         # sidekiq | resque | good_job | cloudtasker | delayed_job | other
auth_strategy: warden_jwt        # devise | warden | jwt | clearance | doorkeeper | custom
money_columns:
  - transactions.amount_cents
  - payouts.amount
critical_paths:
  - app/services/payments/
  - app/controllers/webhooks/
  - app/services/identity_platform/
ignore_paths:
  - app/admin/
  - lib/legacy/
```

When missing, the skill auto-detects from `Gemfile`, workflow filenames, and directory structure.

## Tools used

The skill **invokes** these — it does not re-implement detection. Install whichever you want active; the skill notes any that are missing in the report's appendix.

**Required:**
- [`bundler-audit`](https://github.com/rubysec/bundler-audit) — CVE scan
- [`brakeman`](https://github.com/presidentbeef/brakeman) — Rails SAST
- [`rubocop`](https://github.com/rubocop/rubocop) — style + bug-prone patterns

**Recommended:**
- [`reek`](https://github.com/troessner/reek) — Ruby smells
- [`rails_best_practices`](https://github.com/flyerhzm/rails_best_practices) — Rails-specific antipatterns
- [`flog`](https://github.com/seattlerb/flog) — ABC complexity
- [`flay`](https://github.com/seattlerb/flay) — duplication
- [`simplecov`](https://github.com/simplecov-ruby/simplecov) — coverage data

**Optional / deep mode:**
- [`rubycritic`](https://github.com/whitesmith/rubycritic) — composite quality grade
- [`fasterer`](https://github.com/DamirSvrtan/fasterer) — performance smells
- [`debride`](https://github.com/seattlerb/debride) — dead code
- [`mutant`](https://github.com/mbj/mutant) or [`mutest`](https://github.com/mutest/mutest) — mutation testing

## Severity rubric

The skill applies a strict 4-tier rubric:

- **Blocker** — data loss, money loss, ATO, deploy failure, or compliance violation possible *today*
- **High** — not exploitable/breaking today, but one mistake away
- **Medium** — correctness/perf risk under load or growth
- **Low** — hygiene & maintainability

See [`skills/rails-audit/rubric.md`](skills/rails-audit/rubric.md) for full definitions and calibration examples.

## What it deliberately doesn't do

- **Doesn't re-run RuboCop's job.** Style offenses appear as a count in the appendix, never as findings.
- **Doesn't fix anything.** Audits are read-only. Fixes are a separate session.
- **Doesn't grade product/UX/legal.** A11y, SEO, cookie banners, ToS — different skill.
- **Doesn't claim absolute coverage.** A static audit cannot prove the absence of bugs. The skill says so explicitly when tools are missing.

## Sample output

See [`skills/rails-audit/examples/sample-report.md`](skills/rails-audit/examples/sample-report.md) for a redacted real-world report.

## Contributing

Issues and PRs welcome — especially:
- Additional dimension checks (cite the failure mode in production they'd catch)
- New stack profiles (Kamal, Fly, ECS specifics)
- Tighter severity calibration examples

## License

MIT — see [LICENSE](LICENSE).
