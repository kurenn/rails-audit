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

**Recommended cadence.** `--quick` per PR (or pre-merge), `--standard` quarterly or before a major release, `--deep` once before each significant release. `--continue` is the cheap re-run mode (~10× lower cost) for "I fixed N findings; show me the updated trend."

## Cost & time

The skill calls Anthropic's API through your Claude Code session. Token usage and dollar cost depend on the model your Claude Code is configured with and your project's size. Rough order-of-magnitude estimates from the dogfood projects (30–50 KLOC Rails apps, Sonnet 4.x):

| Mode | Input tokens | Output tokens | Approx. cost (Sonnet 4.5) |
|---|---|---|---|
| `--quick` | ~10–20K | ~3–5K | $0.05–$0.20 |
| `--standard` | ~50–80K | ~10–15K | $0.20–$1.50 |
| `--continue` | ~5K | ~2K | ~$0.05 |
| `--deep` | varies (boots app, runs specs) | varies | $1.00+ |

These are estimates — your actual cost depends on model choice (Opus is ~5× Sonnet), project size, and finding density. The skill writes `cost.estimated_input_tokens` / `cost.estimated_output_tokens` into the JSON report, and when your harness exposes per-call usage metadata it also populates `cost.actual_*`. Watch the first run on a small `--quick` to anchor your own cost expectations before going `--standard` on a 100 KLOC monolith.

## Privacy

**What stays on your machine:**
- Your codebase (the skill never uploads source files).
- Static-tool outputs (`brakeman`, `bundle-audit`, `rubocop`, etc.) — invoked locally via your project's gems.
- The generated report (`tmp/rails-audit/report-*.json`/`*.md`) — written locally, not uploaded.

**What leaves your machine:**
- Findings, file paths, line numbers, and code snippets cited as evidence are sent to Anthropic's API as part of the synthesis prompts (the same way Claude Code already sends file content when you ask it to read files).
- No traffic to third-party services beyond Anthropic.

**Cached:**
- Anthropic's prompt cache may retain prompts/responses per its data-handling terms. If you're auditing a private codebase covered by a vendor agreement, check that your Claude Code's API contract allows it.

The skill itself has no telemetry, no analytics, no "phone home." It runs entirely through your existing Claude Code session.

## Requirements

| | Supported | Notes |
|---|---|---|
| **Rails** | 6.1, 7.x, 8.x | Tested on 7.0.4 (influapp), 7.x (coba), 8.1.3 (pouch). Earlier versions may work but aren't tested. |
| **Ruby** | 3.0+ | Tested on 3.1.2 and 3.4.7. Some Tier-2 gems (`reek`) require 3.4+ for transitive deps. |
| **Test framework** | RSpec, Minitest | RSpec is the primary target; Minitest support is partial (coverage scoring works; spec-stability dimension defaults to RSpec idioms). |
| **Deploy target** | Cloud Run, Heroku, Kamal, Fly, ECS | Detection works for most; some deploy-and-ci checks are Cloud-Run-specific (clearly marked in dimension files). |
| **Auth strategies** | Devise, custom JWT, Identity Platform | The authorization dimension is strategy-agnostic; recommendations are tailored once Step 1 detects which one you use. |
| **Ruby gems** (audited project) | `brakeman`, `bundler-audit` recommended; `rubocop`, `reek`, `rails_best_practices`, `simplecov`, `bullet` enrich coverage | Run `bin/check-tools` to see what's installed in your project. Missing tools degrade output gracefully, not silently. |

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

## Quickstart

A 5-minute first run, end-to-end:

**1. Pre-flight your tooling.** From your Rails project root:

```bash
bin/check-tools
```

That prints a table of which audit tools are installed (`brakeman`, `bundler-audit`, `rubocop`, etc.) and which are missing. Missing Tier-1 tools degrade your first audit's coverage — install them before continuing:

```ruby
# Gemfile
group :development do
  gem 'brakeman', require: false
  gem 'bundler-audit', require: false
  gem 'rubocop', require: false
end
```

Run `bundle install` and re-run `bin/check-tools` until the Tier-1 row is full.

**2. Run your first audit (use `--quick` to start small).** In Claude Code, with the Rails project as your working directory:

```
/rails-audit --quick
```

This finishes in ~3 minutes and costs around $0.05–$0.20 in Sonnet tokens. The skill writes:

- `tmp/rails-audit/report-YYYY-MM-DD.json` — structured source of truth
- `tmp/rails-audit/report-YYYY-MM-DD.md` — rendered report

…and replies with a ≤200-word summary linking to both files.

**3. Read the report.** Open the markdown file. The structure is: executive summary → top blockers → per-dimension scorecards → punch list grouped by Blocker / High / Medium / Low → recommended fix sequence in phases.

**4. Decide what to fix first.** The fix sequence groups findings into phases such that each phase unblocks the next. Phase 1 is usually 1–2 days of work and removes the highest-priority blockers.

**5. (Optional) File issues.** Once you've reviewed the report and want to track the work in GitHub:

```bash
bin/file-issues tmp/rails-audit/report-YYYY-MM-DD.json --mode=per-phase --no-dry-run --update-report
```

Files one issue per phase with the findings as a checklist. Re-running is idempotent (fingerprint-based).

**6. Re-run after fixing.** When you've shipped a few fixes:

```
/rails-audit --continue
```

Skips the static tools + agent fan-out, re-runs only Steps 4.4–7 against the prior report. ~10× cheaper, gives you trend data.

### Going deeper

For the full audit (15 min, ~$0.20–$1.50):

```
/rails-audit
```

For pre-release deep analysis (30+ min, app boots and specs run):

```
/rails-audit --deep
```

To narrow scope:

```
/rails-audit --only=money,security      # only the money + security dimensions
/rails-audit --only-cluster=A,D         # spec-and-coverage + security-and-money clusters
```

See **Project profile** below for opt-in `.audit-config.yml` settings (custom roster, ignore rules, Tier-3 tools).

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

## Calibration status — honest version

The audit pipeline (workflow, dimensions, schema) and the supporting skill versions (v0.2 → v0.5.x) have been built and tested across **three real Rails projects** (influapp + coba + pouch). What that does and doesn't mean:

**What's well-supported by real data**

- The **schema is stable** across v0.2 → v0.5 (4+ minor versions of additive changes; v0.2 reports validate clean against the current schema).
- The **parsers** (`bin/parse-brakeman`, `bin/parse-bundle-audit`, `bin/parse-rubocop`, `bin/parse-reek`, `bin/parse-rails-best-practices`, `bin/scan-secrets`) have a fixture-backed test suite (`bin/test-parsers`) that catches bugs like the `Regexp.last_match`-clobber issue documented in `docs/lessons-learned.md`. CI runs the suite on every PR.
- **Severity inheritance** for EOL CVEs has been validated on influapp (51 of 64 bundle-audit findings demoted; `blocker_pct` dropped from 26.8% → 11.3%) and the inheritance roster has been deliberately conservative (only Rails-bundled and Ruby-adjacent gems).

**N=3 calibration result (pouch, 2026-05)**

Pouch (Rails 8.1.3 / Ruby 3.4.7 / Cloud Run / Devise+JWT) was the first project audited where the calibration *was not* tuned to the codebase in advance. The gates discriminated correctly on data they hadn't seen:

- **C3 (unverified-blocker)** caught a real agent hallucination — a security-cluster agent claimed `.env` was tracked in git; `git ls-files` showed it wasn't. The finding stayed in `unverified_blockers[]`.
- **C7 (calibration override)** correctly did **not** fire — only 5 dimensions scored ≤4 (threshold ≥6). Pouch's profile was concentrated breakage, not broad systemic decay. C7's design intent (discriminate the two) held on novel data.
- **C1 (severity inflation)** fired at the boundary (41.4% high vs 40% threshold). With C7 not applying, the hardened P7 prompt path is the right call.
- **`blocker_pct` = 20.7%** sat well under the v0.5-bumped C2 threshold (30%, raised from 25% specifically to absorb projects shaped like coba).

See `docs/lessons-learned.md` §11 for the post-mortem.

**What's still tuned with the answer in hand**

- The **C7 ≥6-dimensions-≤4 threshold** was chosen so influapp would clear and coba would block. Pouch (5 dims ≤4) sat just below the threshold and correctly didn't fire — that's discrimination, but it's also one boundary data point. A fourth project where C7 fires for the first time on novel data is the next thing that would meaningfully harden the threshold.
- The **C2 30% threshold** was bumped from 25% based on coba's evidence. Pouch came in at 20.7% — well clear. We haven't seen a project at 35% real blockers yet.
- The **token-cost heuristic** in `dimensions/cost-estimation.md` still has worked-example calibration only on the influapp v0.1 actual run (~52K input).

**What hasn't been observed firing on a real run**

Several v0.4-v0.5 features are documented contracts but were not exercised end-to-end against an actual `/rails-audit --standard` invocation in this project's history:

- The hardened block (P7 `block`/`demote`/`accept` flow) — pouch's audit produced the conditions that would trigger it (C1 at boundary, C7 not applying); the block has not yet been observed firing in a live skill invocation through the harness.
- `--continue` / `--from-findings` mode (the cited "10× cost reduction" is an estimate, not a measurement)
- `--track-renames` for trend
- Multi-file output (the 30 KB threshold has not been crossed by any real audit yet)
- The pre-commit hook from `bin/install-hooks` (smoke-tested via `--dry-run` only)

**What that means for you**

If you're using this skill on a project: treat the v0.5 calibration thresholds as a starting point. Run the audit, look at the self-check output, and if C1/C7 fires in a way that feels wrong (e.g., your project genuinely has 8 dimensions ≤4 but the override doesn't apply), open an issue with your `report.json` attached. That's how the thresholds get to 1.0-worthy state.

The `docs/lessons-learned.md` doc captures more of the build's hard-won insights, including the bugs that were caught and the design questions that didn't have a single right answer.

## Troubleshooting

When something goes wrong on a real run, see [`docs/troubleshooting.md`](docs/troubleshooting.md). The most common issues:

- **`bundle exec brakeman` exits non-zero** → brakeman not in your Gemfile, or version mismatch with your Rails version.
- **`gh: command not found` when running `bin/file-issues`** → install `gh` (`brew install gh` on macOS) and authenticate (`gh auth login`).
- **Skill prompt hits a context limit on `--standard`** → the synthesizer ran out of context. Use `--quick` or scope with `--only=<cluster>` instead.
- **`reek` fails with `Ruby 3.4+ required`** → `reek` is Tier-2 (recommended, not required). The audit completes without it; the gap surfaces as a tooling note in the report.
- **`coverage/.resultset.json` is older than 30 days** → C6 self-check fires. Either run your spec suite to refresh it, or accept that test-coverage findings will be marked unverified.

Found an issue not covered there? Open one at [github.com/kurenn/rails-audit/issues](https://github.com/kurenn/rails-audit/issues) with the `report.json` attached if relevant.

## Contributing

Issues and PRs welcome — especially:
- Additional dimension checks (cite the failure mode in production they'd catch)
- New stack profiles (Kamal, Fly, ECS specifics)
- Tighter severity calibration examples

## License

MIT — see [LICENSE](LICENSE).
