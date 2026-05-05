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
