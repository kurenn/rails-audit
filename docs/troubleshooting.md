# Troubleshooting

Common issues hitting a `/rails-audit` run, ordered by how often they bite. If yours isn't here, open an issue at [github.com/kurenn/rails-audit/issues](https://github.com/kurenn/rails-audit/issues) — attach the JSON report if you have one.

## Tooling not detected / missing

### "Tier-1 tools missing" message after `bin/check-tools`

You're missing `brakeman` or `bundler-audit`. Both are required for a useful audit. Add them to your Gemfile under the `:development` group:

```ruby
group :development do
  gem 'brakeman', require: false
  gem 'bundler-audit', require: false
end
```

Run `bundle install`, then re-run `bin/check-tools` to confirm.

### `bundle exec brakeman` exits non-zero

Three usual causes:

1. **Brakeman not actually installed.** `bundle list | grep brakeman` — if empty, see above.
2. **Version mismatch with your Rails.** Brakeman ≥ 7 requires Rails ≥ 6.1; brakeman 5.x covers older Rails. Pin to a compatible version: `gem 'brakeman', '~> 7.0'` (or `~> 5.4` for Rails 5.x).
3. **Project structure brakeman doesn't recognize.** If you have a non-standard layout (engine-only, monorepo subdirectory), pass `-p path/to/rails/root` — but the skill expects to invoke brakeman from the project root, so consider running the audit from there.

### `reek` fails with "Ruby 3.4+ required"

The `reek` gem's transitive dep `dry-schema 1.14.1+` uses Ruby 3.4 anonymous block params. If you're on 3.1 or 3.2, `reek` will fail to boot.

This is **not** an audit blocker — `reek` is Tier-2 (recommended, not required). The audit completes without it and surfaces a tooling note in the report. To get `reek` working, either upgrade Ruby (recommended) or pin `reek` and `dry-schema` to older versions:

```ruby
gem 'reek', '~> 6.1'
gem 'dry-schema', '< 1.14'
```

### `bin/check-tools` says a gem is installed but the audit can't find it

Likely a `BUNDLE_PATH` issue — the skill invokes tools via `bundle exec <tool>`. If you've configured `bundle config path` to a non-default location, `bundle exec` should still resolve correctly, but ensure `bundle install` has run since the last config change.

## `bin/file-issues` fails

### "gh: command not found"

Install the GitHub CLI:

```bash
brew install gh           # macOS
sudo apt install gh       # Debian/Ubuntu
# Or see: https://cli.github.com
```

Then authenticate:

```bash
gh auth login
```

### "could not infer repo from cwd's git remote"

Your project's git remote isn't a github.com URL (e.g., it's a self-hosted GitHub Enterprise or no remote is set). Pass the target explicitly:

```bash
bin/file-issues tmp/rails-audit/report-*.json --repo=owner/repo --mode=per-phase
```

### Issues filed twice / idempotency broken

`bin/file-issues` matches existing issues by a fingerprint marker in the issue body (`<!-- rails-audit:fingerprint=f-... -->`). If you've manually edited the body and removed the marker, re-running creates a new issue. Either:

- Restore the marker in the existing issue's body, or
- Close the duplicate.

### Filed issues are missing labels or milestone

The script tries to create the labels (`audit`, `audit:blocker`, `audit:phase-1`, etc.) on first run via `gh label create`. If the labels exist already with different colors/descriptions, gh skips creation but uses what's there. If you want a `--milestone`, pass it explicitly: `--milestone="Audit Phase 1"`.

## Skill-prompt issues

### Hits a context limit on `--standard`

The 4-cluster fan-out plus synthesis can run long on a 100 KLOC project. Two recovery paths:

1. **Scope down.** `/rails-audit --only-cluster=D` (just security-and-money), or `/rails-audit --only=foundation,deploy-and-ci,security-and-authz` for the deploy-safety triad.
2. **Use `--quick`.** Single-agent flow, no fan-out, much smaller context window.

Or upgrade your Claude Code to a model with a larger context (Sonnet 4.6+).

### Skill blocks at Step 5.5 self-check

This is by design (since v0.4): when C1 (>40% high) or C2 (>30% blocker) fires AND C7's override doesn't apply, the skill stops and asks you to pick one of:

- `block` — don't ship this report; revisit findings (the default).
- `demote` — auto-demote inflated severities one tier, then proceed.
- `accept` — proceed with the warnings noted; an override entry lands in `audit.calibration_overrides[]`.

If you didn't expect this and want to disable the block, see `dimensions/self-check.md` for the rules' details. Most of the time, the right move is `block` and revisit — the skill is telling you something.

### Synthesizer wrote `audit.commit = "HEAD"` (or other invalid values)

Caught at Step 7 by `bin/validate-report`. The validator surfaces a friendly error pointing at the fix (`git rev-parse --short=12 HEAD`). If the validator's still reporting an issue you can't decode, paste its stderr output into a GitHub issue.

## Coverage / SimpleCov

### `coverage/.resultset.json` older than 30 days → C6 fires

This is the v0.3 self-check at work. Either:

1. **Refresh coverage.** Run your spec suite (`bundle exec rspec` or whatever your project uses) so SimpleCov writes a fresh `coverage/.resultset.json`.
2. **Accept the warning.** Test-coverage findings are marked `self_check.status: "unverified"` — they're not removed, but the report is honest about the freshness. If your project doesn't run SimpleCov in CI, this might be permanent; consider adding a coverage job.

### Test-coverage scores are 0% for every dimension

Either you don't have SimpleCov configured, or `coverage/.resultset.json` doesn't exist. Add SimpleCov:

```ruby
# Gemfile
group :test do
  gem 'simplecov', require: false
end
```

```ruby
# spec/spec_helper.rb (or test_helper.rb), at the very top
require 'simplecov'
SimpleCov.start 'rails'
```

Run your suite once, then re-audit.

## Report rendering / output

### `tmp/rails-audit/` doesn't exist or is empty

Step 7 writes to that directory. If it's missing, the audit either failed before Step 7 or your project's `.gitignore` is excluding `tmp/` (Rails default — that's fine; the report just isn't checked into git). Check the Claude Code conversation log for any error messages from Step 7.

### Report markdown looks mangled / template variables not interpolated

The renderer uses `output-template.md` as the source of truth; if you've forked this skill and modified the template, ensure all placeholders match the JSON schema's field paths. The template format is documented in the file's own header.

### Multi-file mode kicked in when I expected single-file

The skill auto-switches to multi-file mode when the rendered markdown crosses 30 KB. Force single-file with `/rails-audit --single-file` if your viewer prefers it.

## Plugin install / version

### `claude plugin install rails-audit@kurenn` fails

1. Ensure the marketplace is added: `claude plugin marketplace add kurenn/marketplace`.
2. Refresh the cache: `claude plugin marketplace update kurenn`.
3. Try install again. If still failing, the [marketplace.json](https://github.com/kurenn/marketplace/blob/main/.claude-plugin/marketplace.json) might list a stale source URL — open an issue.

### Slash menu still doesn't show `/rails-audit` after install

Restart your Claude Code session. The slash menu loads at startup.

### Skill is installed but I want to pin to an older version

Clone the tag and load via `--plugin-dir`:

```bash
git clone https://github.com/kurenn/rails-audit --branch v0.5.0 ~/workspace/rails-audit-0.5.0
claude --plugin-dir ~/workspace/rails-audit-0.5.0
```

## Calibration / self-check questions

### "Why did C7 not fire on my project? It's clearly on fire."

C7 requires **all three** conditions: every blocker is sed-verified (no hallucinations), `summary.risk_score ≤ 4`, and ≥6 dimensions scored ≤4. The most common reason it doesn't apply: **fewer than 6 dimensions ≤4**. That's by design — concentrated breakage (3–4 dims on fire) is treated differently from systemic decay (≥6 dims on fire).

If you genuinely think the threshold is wrong for your shape of project, open an issue with the report.json — that's how the calibration arc gets to N=4.

### "C1 fires every audit. Is the threshold wrong?"

Possibly, but more likely the synthesizer is defaulting to "high" on borderline findings. Re-anchor against `skills/rails-audit/rubric.md`:

- **High** = "exploit/loss/breakage path with one mitigating layer between attacker and impact."
- **Medium** = "wrong but contained" — a defect that won't ship a CVE but is the kind of thing reviewers want fixed.

If after re-anchoring you still see >40% high, that's calibration evidence. Open an issue.

## Last resort

If nothing here matches, capture the failure with:

```bash
bin/check-tools --json > /tmp/audit-debug-tooling.json
cat tmp/rails-audit/report-*.json   # if one exists
```

…and open an issue at [github.com/kurenn/rails-audit/issues](https://github.com/kurenn/rails-audit/issues) with both files attached.
