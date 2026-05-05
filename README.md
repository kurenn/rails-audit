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

Clone into your Claude Code skills directory:

```bash
git clone https://github.com/kurenn/rails-audit.git ~/.claude/skills/rails-audit
```

Or as a project-scoped skill:

```bash
git clone https://github.com/kurenn/rails-audit.git .claude/skills/rails-audit
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

Run from any Rails project root with the skill installed at `~/.claude/skills/rails-audit`. Reads `tooling.md` for tier definitions, then detects each tool by Gemfile presence + binary in PATH.

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

See [`rubric.md`](rubric.md) for full definitions and calibration examples.

## What it deliberately doesn't do

- **Doesn't re-run RuboCop's job.** Style offenses appear as a count in the appendix, never as findings.
- **Doesn't fix anything.** Audits are read-only. Fixes are a separate session.
- **Doesn't grade product/UX/legal.** A11y, SEO, cookie banners, ToS — different skill.
- **Doesn't claim absolute coverage.** A static audit cannot prove the absence of bugs. The skill says so explicitly when tools are missing.

## Sample output

See [`examples/sample-report.md`](examples/sample-report.md) for a redacted real-world report.

## Contributing

Issues and PRs welcome — especially:
- Additional dimension checks (cite the failure mode in production they'd catch)
- New stack profiles (Kamal, Fly, ECS specifics)
- Tighter severity calibration examples

## License

MIT — see [LICENSE](LICENSE).
