# Changelog

All notable changes to this skill are documented in this file. Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and the project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added (v0.3 milestone PR#1 — Calibration bundle: C6 stale coverage + C7 real-distribution override + cost constants)

> Note: the v0.3 milestone is tracked under that label but ships as **v0.4.0** since v0.3.0 was already used for the plugin-restructure release.

- **C6 — Stale coverage data** in `skills/rails-audit/dimensions/self-check.md`. Warns when `coverage/.last_run.json` or `.resultset.json` mtime is >30 days old. Affected `test-coverage` findings auto-marked `self_check.status: "unverified"`. Both v0.2 dogfood projects (influapp, coba) had Mar-2025 coverage data — would have fired this check.
- **C7 — Real-distribution override** in `skills/rails-audit/dimensions/self-check.md`. When C1 (severity inflation) or C2 (blocker over-use) would fire, suppress them if all blockers are sed-verified AND `summary.risk_score ≤ 4` AND ≥6 dimensions score ≤4. Distinguishes "project-on-fire" from "synthesis-inflated." Emits its own diagnostic message.
- **Cost calibration update** in `skills/rails-audit/dimensions/cost-estimation.md`:
  - `per_kloc_in`: 200 → 300 (based on influapp v0.1 underestimate)
  - New: `per_kloc_lib` = 100 (non-`app/` code factor — coba has Monex bindings under `lib/`)
  - New: `per_kloc_eng` = 50 (engines/* factor, typically 0)
  - Worked examples updated; influapp MAPE drops from ~16% to ~13%.
- Closes #27 (C6), #28 (C7), #37 (cost calibration).

## [0.3.0] — 2026-05-05

First release as an installable Claude Code plugin. Audit content unchanged from 0.2.0.

### Added

- `.claude-plugin/plugin.json` manifest. Install via marketplace:
  ```bash
  claude plugin marketplace add kurenn/marketplace
  claude plugin install rails-audit@kurenn
  ```

### Changed

- Repo restructured into plugin layout: `SKILL.md` and all referenced files
  (`dimensions/`, `schema/`, `examples/`, `prompts.md`, `rubric.md`,
  `tooling.md`, `output-template.md`) moved under `skills/rails-audit/`.
- `bin/`, `README.md`, `CHANGELOG.md`, `LICENSE`, and `.github/` stay at repo root.
- Internal references in `SKILL.md` keep their relative paths and continue to work.

### Migration

If you were copying `SKILL.md` into `~/.claude/skills/rails-audit/` by hand,
switch to the marketplace install above. The standalone install pattern still
works — copy `skills/rails-audit/` (the new path) instead of the repo root.

## [0.2.0] — 2026-05-05

Major refinement informed by the v0.1 dogfood against influapp-api. JSON becomes the source of truth for reports; markdown is rendered from it. The skill now polices its own output (self-check), supports finding-level trend tracking, scoped audits, and acknowledged-finding suppression. 11 PRs across 4 phases.

### Added

- **JSON-first synthesis** — `schema/report.schema.json` is the contract; `output-template.md` is a render. Stable finding fingerprints (`SHA256(primary_dimension + file_path + finding_type + normalize(evidence_snippet))`). [#13]
- **Cross-dimension finding tags** — `secondary_dimensions[]` populated by synthesis; per-dimension scorecards count findings where the dimension appears in `primary_dimension` OR `secondary_dimensions[]`. Each `dimensions/*.md` documents its known cross-cuts. [#14]
- **`prompts.md`** — every user-facing question (P1–P7) documented with answers, defaults, and fallbacks. [#15]
- **`bin/check-tools`** — standalone Ruby script (no gem deps) inventorying audit tooling. Modes: human-readable, `--json`, `--required-only` (CI exit code), `--help`. [#16]
- **Audit-the-audit self-check (Step 5.5)** — five checks (severity inflation, blocker over-use, unverified blocker, phase dependency mismatch, scorecard mismatch). Warn-only in v0.2; hardens to block in v0.3. [#17]
- **Money-path re-validation pass (Step 4.5)** — focused second pass on every money-tagged finding. Six checks (M-RV-1 idempotency-key derivation, M-RV-2 transaction-boundary ordering, M-RV-3 webhook event-ID dedup, M-RV-4 money column types, M-RV-5 refund idempotency, M-RV-6 audit trail). Statuses: `confirmed` / `refined` / `rejected` / `promoted`. [#18]
- **`.audit-ignore.yml`** — fingerprint-keyed suppression of acknowledged findings. Required: `id`, `reason`. Optional: `acknowledged_by`, `expires_at`. Stale ignores surface in `audit.ignore_warnings[]`; expired ignores re-surface with a marker. Applied at Step 4.4. [#19]
- **`--only` / `--exclude` scoped audits** — narrow an audit to specific dimensions or aliases. 17 aliases (`money`, `security`, `auth`, `deploy`, `specs`, `coverage`, `perf`, `code`, `jobs`, `obs`, `data`, `dx`, `cost`, etc.). Mutually exclusive with each other; Levenshtein-suggested correction on typos. [#20]
- **Token-budget awareness (Step 0.5)** — pre-flight estimate (`mode_mult * (5000 + 2000*dims + 200*kloc_app)` for input, `* (3000 + 600*dims + 10*kloc_app)` for output). Prompt P5 fires above 30K input. `--budget=<N>` enforces a cap with partial-report fallback. [#21]
- **Finding-level trend (Step 5.7)** — replaces v0.1's score-deltas with per-finding diff: fixed / new / persisted, with first-seen propagation. Per-dimension diff table. Stale-persisted sub-table for findings ≥30 days old. Empty trend (first run) is omitted entirely. [#22]
- **Reference points per dimension** — each of the 12 primary dimension files closes with a `## Reference points` section citing 3–6 OSS Rails projects, gems, or official docs (Solidus, Discourse, Mastodon, GitLab, Rails Security Guide, strong_migrations, lockbox, Suspenders, etc.). [#23]
- New supporting dimension files: `dimensions/self-check.md`, `dimensions/money-revalidation.md`, `dimensions/cost-estimation.md`, `dimensions/trend-tracking.md`.

### Changed

- **`SKILL.md`** — workflow expanded from 7 to 11 steps: Step 0 (provision) + 0.5 (estimate) + 1 (detect) + 2 (static tools) + 3 (fan-out) + 4 (synthesize → JSON) + 4.4 (apply ignore) + 4.5 (money revalidate) + 5.5 (self-check) + 5.7 (trend) + 6 (render) + 7 (write) + 8 (brief).
- **`output-template.md`** — refactored from prose into a JSON-driven render template with named filters (`date`, `dimension_label`, `range_str`, `percent`, `join`, `where`, `sort_by`, plus 8 trend-specific filters added in [#22]).
- **`examples/sample-report.{json,md}`** — regenerated to demonstrate every v0.2 feature: cross-dimension tags, money revalidation (`confirmed`/`refined`/`promoted` examples), an ignored finding, a stale-ignore warning, self-check calibration warnings, and an unverified blocker.

### Migration from v0.1

- Reports written by v0.1 are markdown-only; they don't carry fingerprints. Trend tracking only works between v0.2+ JSON reports.
- The Gemfile additions in v0.1 (Step 0 provisioning) are still recommended; v0.2 doesn't change which tools are needed, only how the skill orchestrates them.

### Locked-in design decisions (recorded for future-self)

- **Fingerprint algorithm**: stable across line moves + whitespace; unstable across renames + material code changes. Renames intentionally produce "fixed + new" in trend.
- **JSON-first**: schema is the contract; markdown is one of potentially several views.
- **Self-check warns, doesn't block (v0.2)**: hardens to block in v0.3 once thresholds are calibrated against ~5 real audits.

## [0.1.0] — 2026-05-04

Initial release.

### Added

- **`SKILL.md`** — entry point with three modes (`--quick`, `--standard`, `--deep`), a 6-step workflow plus Step 0 for tooling provisioning, and a project-profile mechanism (`.claude/rails-audit.yml`) for per-project overrides.
- **`rubric.md`** — strict 4-tier severity rubric (Blocker / High / Medium / Low) with explicit calibration examples and anti-patterns.
- **`tooling.md`** — tool contract (Tier 1 required, Tier 2 recommended, Tier 3 optional), provisioning policy, copy-pasteable Gemfile snippet, and adapter detection (job adapter, auth strategy, deploy target).
- **`output-template.md`** — exact report skeleton with executive summary, per-dimension scorecards, severity-grouped punch list, recommended fix sequence, and trend table.
- **`dimensions/`** — 12 cluster files covering 18 dimensions:
  - `domain-shape.md`
  - `spec-and-coverage.md`
  - `deploy-ci.md`
  - `security-and-authz.md`
  - `money-and-payments.md`
  - `code-health.md`
  - `performance-reliability.md`
  - `background-jobs.md`
  - `observability.md`
  - `data-integrity.md`
  - `data-governance.md`
  - `dx-and-cost.md`
- **`examples/sample-report.md`** — illustrative report based on a real audit.
- **`README.md`** — installation and usage; **`LICENSE`** — MIT.

### Workflow

- 4-cluster parallel agent fan-out (Spec/Coverage, Deploy/CI/Obs, Code Health, Security/Money) for `--standard` mode.
- Static tool invocation pattern: skill orchestrates `brakeman`, `bundler-audit`, `rubocop`, `reek`, `rails_best_practices`, `flog`, `flay`, `simplecov`, `rubycritic` and synthesizes findings — never re-implements detection.
- Severity-first synthesis: deduplicate across clusters, apply rubric strictly, sequence fixes so each phase unblocks the next.

[Unreleased]: https://github.com/kurenn/rails-audit/compare/v0.2.0...HEAD
[0.2.0]: https://github.com/kurenn/rails-audit/releases/tag/v0.2.0
[0.1.0]: https://github.com/kurenn/rails-audit/releases/tag/v0.1.0
