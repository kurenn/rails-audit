# Changelog

All notable changes to this skill are documented in this file. Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and the project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added (v0.2 PR#1 — Schema + JSON-first synthesis)

- **`schema/report.schema.json`** — JSON Schema (draft 2020-12) defining the report contract. Mandatory fields: `schema_version` (=2), `skill_version`, `audit{}`, `summary{}`, `scorecards[]`, `findings[]`, `tooling{}`. Optional: `ignored_findings[]`, `trend{}`, `self_check{}`, `cost{}`, `fix_sequence[]`, `appendices{}`.
- **`examples/sample-report.json`** — the influapp dogfood report rendered into the new schema (25 findings, 18 scorecards, 5 phases). Validates against the schema.
- Stable finding fingerprints: `f-` + first 16 hex of `SHA256(primary_dimension + file_path + finding_type + normalize(evidence_snippet))`. `normalize` strips comments, collapses whitespace, preserves case.

### Changed

- **`SKILL.md`** — Step 4 (synthesize) now produces JSON conforming to `schema/report.schema.json`. New Step 5 renders markdown from the JSON via `output-template.md`. New Step 6 writes both `.json` and `.md` to `tmp/rails-audit/`.
- **`output-template.md`** — refactored from free-form prose into a JSON-driven render template. Placeholder names (`{{json.path}}`) refer to JSON paths. Filters (`date`, `dimension_label`, `range_str`, `percent`, `join`, `where`, `sort_by`) are deterministic.
- **`examples/sample-report.md`** — regenerated as the rendered output of `sample-report.json`. Notes at the top: edits should land in the JSON or the template, not directly in this file.

### Migration

- Reports written by v0.1 of the skill are markdown-only; they don't carry fingerprints. Trend tracking (planned PR#10) will only work between v0.2+ JSON reports.

### Added (v0.2 PR#9 — Token-budget awareness)

- **`dimensions/cost-estimation.md`** — token-cost heuristic with worked examples. Pre-flight estimate based on mode multiplier × (base + per-dimension constant + per-KLOC constant for `app/`). Calibrated to ~30% MAPE on standard-mode runs of typical Rails projects (~10K–50K LOC).
- **`SKILL.md`** new **Step 0.5** between provisioning and detect: compute estimate; trigger prompt **P5** if `--budget` is unset and estimated input > 30K tokens.
- **`--budget=<N>`** flag enforced: each agent call is gated on remaining budget; budget hit saves partial JSON + flagged-partial markdown rather than silently truncating.
- **Hard floor**: `--budget=<N>` where N < 5000 is rejected.
- Schema `cost{}` and template Appendix E already supported these fields from PR#1; this PR populates them.

### Added (v0.2 PR#8 — `--only` / `--exclude` scoped audits)

- **`--only=<comma-list>`** — run only the named dimensions or aliases. Example: `/rails-audit --only=money,security`.
- **`--exclude=<comma-list>`** — run everything except the named dimensions/aliases.
- **Positional shortcut** — a single non-flag arg is treated as `--only=<arg>`: `/rails-audit money`.
- **Aliases** (case-insensitive): `money` → `money-and-payments`, `security` → `security-and-authz`, `auth`/`authz` → `authorization`, `deploy`/`ci` → `deploy-and-ci`, `specs` → `spec-stability`, `coverage` → `test-coverage`, `perf` → `performance` + `reliability`, `code` → `code-smells` + `risk-hotspots`, `jobs` → `background-jobs`, `obs` → `observability`, `data` → `data-integrity` + `data-governance`, `dx` → `developer-experience`, `cost` → `cost-and-scaling`, `all` → all 18.
- **Validation**: unknown dimension/alias rejected with a Levenshtein-suggested correction; `--only` and `--exclude` are mutually exclusive.
- **Behavior**: scoped audits skip cluster fan-out for irrelevant clusters, filter scorecards and findings to in-scope dimensions (including findings whose `secondary_dimensions[]` overlap), renumber the scorecard table from 1, and gate Step 4.5 (money revalidation) on scope membership.

### Added (v0.2 PR#7 — `.audit-ignore.yml` suppression mechanism)

- **`.audit-ignore.yml`** at repo root suppresses acknowledged findings by fingerprint. Each entry: `id` (required), `reason` (required, non-empty), `acknowledged_by` (optional), `expires_at` (optional ISO date).
- **`SKILL.md`** new **Step 4.4** between synthesis and money re-validation:
  - Sets `findings[<id>].ignored = true` for matching entries.
  - Builds top-level `ignored_findings[]` array.
  - Surfaces stale ignores (no current finding matches) in `audit.ignore_warnings[]`.
  - Re-surfaces expired ignores with a `_(ignore expired YYYY-MM-DD)_` note.
- Findings with `ignored: true` are excluded from punch list, money re-validation, and self-check calibration percentages — but **included** in trend tracking (so an ignored finding still counts as "persisted" if it was in the prior report).
- **`examples/.audit-ignore.yml.example`** documents the format with three illustrative entries: live, permanent (no expiry), and expired.
- **`examples/sample-report.json`**: M4 (No bullet gem) marked `ignored: true` with a corresponding `ignored_findings[]` entry; one stale-ignore (`f-deadbeefdeadbeef`) demonstrates the warning path.
- **`examples/sample-report.md`**: regenerated. Mediums table renumbered (M5 → M4 since old M4 is suppressed); Phase 4 fix sequence drops the suppressed item; Appendix D — Ignored findings populated.
- Schema and template already supported `ignored_findings[]` and `audit.ignore_warnings[]` from PR#1; no schema or template change.

### Added (v0.2 PR#6 — Money-path re-validation pass)

- **`dimensions/money-revalidation.md`** — defines the focused second pass on every finding tagged `money-and-payments` (primary OR secondary). Six money-specific re-checks: M-RV-1 idempotency-key derivation, M-RV-2 transaction boundary ordering, M-RV-3 webhook event-ID dedup, M-RV-4 money column types (Float = blocker), M-RV-5 refund/reversal idempotency, M-RV-6 audit-trail completeness.
- **`SKILL.md`** new **Step 4.5** between synthesis and self-check. Sets `findings[<id>].money_revalidation` to `confirmed` / `refined` / `rejected` / `promoted`. Includes a proactive sweep on `app/services/payments/*`, `app/controllers/webhooks/stripe_*`, `app/jobs/*payment*`, `app/jobs/*payout*`, and money-shaped columns in `db/schema.rb`.
- **`examples/sample-report.json`** — three findings now exercise the field: `confirmed` (webhook dedup), `refined` (update_column on stripe_customer_id), `promoted` (Stripe payouts missing idempotency keys, originally tagged medium).
- **`examples/sample-report.md`** — regenerated to show the `_Re-validated: <status>_` badges.
- Schema and template already supported `money_revalidation` from PR#1; no schema or template change needed.

### Added (v0.2 PR#5 — Audit-the-audit self-check)

- **`dimensions/self-check.md`** — defines five calibration checks the skill runs against its own report before delivery: C1 severity inflation (>40% high), C2 blocker over-use (>25% blocker), C3 unverified blocker (cited line not `sed`-confirmed), C4 phase dependency mismatch, C5 scorecard ↔ finding-count mismatch.
- **`SKILL.md`** new Step 5.5 (between synthesis and render). Runs the five checks; surfaces results in `self_check.calibration.warnings[]` and per-finding `self_check.status`.
- **In v0.2 self-check is warn-only.** No findings are removed or demoted. Hardening to block is planned for v0.3 once thresholds are calibrated against ~5 real audits.
- Step numbering shifted downstream: render is now Step 6, write Step 7, brief Step 8.

### Added (v0.2 PR#4 — `bin/check-tools` standalone command)

- **`bin/check-tools`** — Ruby script that inventories rails-audit tooling in any Rails project.
  - No args: human-readable tier-grouped table with Tool / Status / Version / Install hint.
  - `--json`: machine-readable output matching the `tooling{}` block of `report.schema.json` (so it can be reused inside an audit run).
  - `--required-only`: exits 1 if any Tier-1 tool is missing — useful as a CI pre-flight.
  - `--help`: usage.
- Reads `tooling.md` to extract tier definitions; only parses Tier 1–3 (Tier 4 is operational, not gems).
- ANSI-strip + sanity-check on tool `--version` output to avoid showing error messages as version strings.
- README documents the command.

### Added (v0.2 PR#3 — Formalize interactive prompts)

- **`prompts.md`** — every user-facing question in the skill (P1–P7) documented in one file with question text, accepted answers + aliases, default, fallback, and side-effects.
  - **P1** Mode selection (Quick/Standard/Deep)
  - **P2** Tool provisioning per missing required tool (gemfile/global/skip + hard floor)
  - **P3** Recommended tool batch
  - **P4** Project profile init (first run only)
  - **P5** Pre-fan-out budget confirmation (when budget unset and estimate >30K tokens)
  - **P6** Ignore-finding reason (free-form, with 90-day default expiry)
  - **P7** Self-check warning surfaced to user (show/demote/accept)
- **`SKILL.md`** updated to reference prompts by ID (P1–P4) instead of inlining prompt text. Future PRs will reference P5–P7 as the relevant features land.
- Anti-patterns section in `prompts.md` codifies what to avoid: open-ended without default, stacked questions, hidden side effects, unbounded retries.

### Added (v0.2 PR#2 — Cross-dimension finding tags)

- **`## Cross-cuts` section** added to all 12 dimension files (`spec-and-coverage`, `deploy-ci`, `security-and-authz`, `money-and-payments`, `code-health`, `performance-reliability`, `background-jobs`, `observability`, `data-integrity`, `data-governance`, `dx-and-cost`, `domain-shape`). Each section documents the dimension's known overlaps so synthesis can populate `secondary_dimensions[]` consistently.
- **`SKILL.md` Step 4** updated with explicit instructions for tagging cross-dimensional findings: pick the most-load-bearing dimension as primary, tag all applicable others in `secondary_dimensions[]`. Concrete examples (`update_column(:stripe_customer_id)`, Stripe webhook missing event-ID dedup, `force_ssl` disabled).
- Per-dimension scorecards now meaningfully aggregate findings where the dimension appears in `primary_dimension` OR any `secondary_dimensions[]`.

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

[Unreleased]: https://github.com/kurenn/rails-audit/compare/v0.1.0...HEAD
[0.1.0]: https://github.com/kurenn/rails-audit/releases/tag/v0.1.0
