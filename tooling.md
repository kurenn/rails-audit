# Tooling contract

The skill **invokes** existing tools and **synthesizes** their output. It does not re-implement detection.

## Provisioning policy

The skill enforces tool availability in **Step 0** of its workflow (see `SKILL.md`). Required tools are mandatory; missing them aborts the audit. Recommended tools are batch-prompted. Optional tools are deep-mode only.

**The skill never modifies the Gemfile silently.** It always prompts and shows the exact diff before applying. If the user prefers to install manually, the snippet below is copy-pasteable.

## Recommended Gemfile additions

Paste this into your `Gemfile` (the skill will offer to do it for you in Step 0):

```ruby
group :development do
  # Required: security SAST and dependency CVE scanning
  gem 'brakeman', require: false
  gem 'bundler-audit', require: false

  # Recommended: code smells and Rails-specific antipatterns
  gem 'reek', require: false
  gem 'rails_best_practices', require: false

  # Recommended: complexity + duplication
  gem 'flog', require: false
  gem 'flay', require: false

  # Optional: composite quality grade per file
  gem 'rubycritic', require: false

  # Optional: performance smells / dead code
  gem 'fasterer', require: false
  gem 'debride', require: false
end

group :development, :test do
  # Recommended: runtime N+1 detection (and other AR smells)
  gem 'bullet'
end

# Already common in many Rails projects:
# gem 'rubocop', require: false
# gem 'simplecov', require: false
```

Then:

```bash
bundle install
# Optional: generate binstubs for faster invocation
bundle binstubs brakeman bundler-audit reek rails_best_practices flog flay
```

## Tier 1: required

## Tier 1: required

These should be available in any reasonably maintained Rails project. If missing, install warning in the report.

| Tool | Invoke | Output | Why |
|---|---|---|---|
| `bundler-audit` | `bundle exec bundle-audit check --update --format json` (or `bundle audit`) | `tmp/rails-audit/bundle-audit.json` | Known CVEs in deps |
| `brakeman` | `bundle exec brakeman -q -f json -o tmp/rails-audit/brakeman.json` | JSON report | SAST for Rails — SQL injection, mass assignment, redirect, XSS |
| `rubocop` | `bundle exec rubocop --format json --out tmp/rails-audit/rubocop.json` | JSON | Style + bug-prone patterns; treat *count by severity* as signal, not individual offenses |
| `rails routes` | `bin/rails routes -E` (or `bundle exec rails routes`) | `tmp/rails-audit/routes.txt` | Endpoint inventory |
| `bundle outdated` | `bundle outdated --strict` | `tmp/rails-audit/outdated.txt` | Lock freshness |

## Tier 2: recommended

| Tool | Invoke | Why |
|---|---|---|
| `reek` | `bundle exec reek -f json app/ lib/` | Generic Ruby smells (duplication, feature envy, long parameter list) |
| `rails_best_practices` | `bundle exec rails_best_practices --format json --output-file tmp/rails-audit/rbp.json .` | Rails-specific antipatterns |
| `flog` | `bundle exec flog -m app/ lib/` | ABC complexity per method; identifies refactor targets |
| `flay` | `bundle exec flay app/ lib/` | Structural duplication |
| `simplecov` results | Read `coverage/.last_run.json` and `coverage/.resultset.json` | Per-file coverage, branch coverage if enabled |

## Tier 3: optional / deep mode

| Tool | Invoke | Why |
|---|---|---|
| `rubycritic` | `bundle exec rubycritic --no-browser --format json` | Composite quality grade per file |
| `fasterer` | `bundle exec fasterer` | Performance smells |
| `debride` | `bundle exec debride app/ lib/` | Dead method detection |
| `mutant` or `mutest` | `bundle exec mutant run --include lib --require my_app -- 'MyApp::Payments*'` | Mutation testing on critical paths only — confirms tests catch regressions |
| `bullet` | Boot app with `Bullet.enable = true` and exercise endpoints | Runtime N+1 detection |
| `pg_query` analysis | `bin/rails db:schema:dump` then read `db/schema.rb` | Index analysis, missing FK indexes |

## Tier 4: external sanity (deep mode only)

| Tool | Invoke | Why |
|---|---|---|
| `bin/setup` | Run on a fresh checkout in a worktree | Onboarding friction signal |
| `bin/rspec --tag '~slow'` | Subset of specs | Confirm suite at least *runs* |
| `docker build .` | Production target only | Image build sanity |

## Detection

Before invoking, detect what's available:

```bash
# Check Gemfile for declared tools
grep -E "^\s*gem ['\"](brakeman|bundler-audit|rubocop|reek|rails_best_practices|rubycritic|simplecov|bullet|fasterer|debride|mutant)['\"]" Gemfile 2>/dev/null
# Check binaries
which brakeman bundle-audit rubocop reek rails_best_practices 2>/dev/null
```

Always prefer `bundle exec <tool>` if the tool is in the Gemfile, otherwise the bare command.

## When tools are missing

In the report's "Tooling" appendix, list:
- What ran (with version captured via `<tool> --version`)
- What was missing
- One-line install hint per missing tool: `gem 'brakeman', group: :development` etc.

Don't hide gaps — a missing `brakeman` means the security dimension's confidence is reduced, and the report should say so.

## Output discipline

- All tool output written to `tmp/rails-audit/`. Add `tmp/rails-audit/` to `.gitignore` if not already covered by `tmp/`.
- Tool output is *evidence*, not the report. Parse the relevant subset and cite specific files/lines in the punch list.
- Don't paste raw tool output into the report body. Summarize counts in the appendix; cite specifics in findings.

## Adapter detection (drives dimension specialization)

```bash
# Job adapter
grep -E "gem ['\"](sidekiq|resque|good_job|cloudtasker|delayed_job)['\"]" Gemfile

# Auth
grep -E "gem ['\"](devise|warden|jwt|clearance|doorkeeper)['\"]" Gemfile

# Deploy target
ls .github/workflows/ 2>/dev/null
test -f fly.toml && echo "fly"
test -f Procfile && grep -i heroku Procfile && echo "heroku"
test -f config/deploy.yml && echo "kamal"
test -f Dockerfile && grep -i "cloud-run\|cloudrun" .github/workflows/*.yml 2>/dev/null && echo "cloud_run"
```

Pass detected stack into agent briefs so dimension checks adapt (e.g. Cloud Run gets a `min-instances` check; Heroku gets a dyno-config check).
