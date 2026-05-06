# Changelog

All notable changes to this skill are documented in this file. Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and the project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.5.2] — 2026-05-06

Patch release that closes the gap surfaced by the pouch dogfood: the synthesizer could write reports containing schema-invalid `audit.commit = "HEAD"` and non-enum `time_estimate` values without catching them before `File.write`. Lessons-learned §12 has the post-mortem.

### Added

- **`bin/validate-report`** — pre-write gate for the report JSON. Wraps `json_schemer` plus extra "synthesis-pattern" sanity checks that surface friendlier errors than raw schema violations:
  - `audit.commit = "HEAD"` → hint to use `git rev-parse --short=12 HEAD`.
  - `time_estimate` not in the enum → hint about the round-up convention (`30m` → `1h`, `2h` → `1d`).
  - Cross-references that point to non-existent finding ids (`fix_sequence.finding_ids`, `summary.top_blocker_ids`, `severity_inherited_from`, `self_check.{verified,unverified}_blockers`).
  - `top_blocker_ids` pointing at non-blocker findings.
  - Stdin-friendly (`bin/validate-report -`) so the synthesizer can pipe in-memory JSON through it before writing to disk.
- **6 new `bin/test-parsers` checks** for `validate-report`: valid minimal report passes, the pouch-shaped failure (HEAD commit + 30m time_estimate) is caught with both schema and sanity tags, cross-ref consistency is enforced. 19 → 25 checks total.

### Changed

- **`SKILL.md` Step 7 (write outputs)** — now mandates running `bin/validate-report -` on the in-memory report before `File.write`. If validation fails, the synthesizer must repair and re-validate; do not write a known-invalid report.

### What didn't change

No schema changes. v0.5.1 and v0.6+ reports both validate under the new gate without modification. Schema additivity discipline preserved.

## [0.5.1] — 2026-05-06

Honest-version patch release. Fixes three classes of gap surfaced by a self-rating exercise.

### Fixed

- **Parser fingerprints didn't match the schema regex.** `bin/parse-brakeman`, `bin/parse-bundle-audit`, and `bin/scan-secrets` were emitting ids like `f-bk-0001-1a52` and `f-secret-0001-96d` — the schema requires `^f-[0-9a-f]{12,64}$` (pure hex). Replaced all three with `f-` + first 16 hex of `SHA256(seed)`. Stable across runs (good for trend); schema-compliant.
- **`Regexp.last_match` clobber regression check.** The bug documented in `docs/lessons-learned.md` is now caught by `bin/test-parsers`'s "record values populated (not nil)" check on `parse-bundle-audit`. Future parsers will fail loudly if they hit the same gotcha.

### Added

- **`bin/test-parsers`** smoke test runner with 19 checks across all 5 parser scripts. Validates: exit 0, valid JSON output, fingerprint regex match, severity tier match, false-positive filtering (Open3 array form), aggregate `total` populated. Fixtures under `test/fixtures/`.
- **`bin/apply-inheritance`** promoted into the repo from `/tmp/apply-inheritance.py` (rewritten in Ruby for consistency with the other `bin/*` scripts). Reproduces the v0.5 dogfood demotion on influapp v0.4 → v0.5: 51 demoted, blocker_pct 26.8% → 11.3%, high_pct 62.9% → 42.3%.

### Documented

- **README.md `## Calibration status — honest version`** section. Loud disclosure that:
  - Schema and parsers are well-tested across 4 minor versions.
  - C7 override thresholds, C2 30% threshold, and inheritance roster were tuned with the influapp + coba data already in hand. N=2 calibration. **A third project may push back on any of those numbers.**
  - The v0.4/v0.5 dogfood reports were synthesized from v0.2 + parser outputs, NOT produced by fresh end-to-end agent runs. The "10× cost reduction" for `--continue` is an estimate, not a measurement.
  - Several v0.4-v0.5 features (hardened block, `--continue`, `--track-renames`, multi-file output, install-hooks pre-commit) are documented contracts but not observed firing on real audits.
- **`docs/lessons-learned.md`** sections 9 and 10: "Calibration on N=2 isn't calibration" and "The dogfood reports were syntheses, not real runs."

### Triaged

- **v0.6 milestone PRs #2-#5 closed as `not planned`** with comment: "deferring until PR#1 (MVP) is dogfooded against a real project." The MVP (#65) stays open as the canonical scope for the next iteration. This reduces the paper-plan maintenance burden noted in the self-rating.

### What didn't change

No behavioral changes to the audit pipeline. Schema is unchanged from 0.5.0. v0.5 reports validate identically.

## [0.5.0] — 2026-05-05

Calibration round 2 driven by the v0.2→v0.4 benchmark on influapp + coba. Six PRs across calibration, parser completion, and pre-commit integration.

### Highlights

- **Severity inheritance for EOL CVEs** — when N findings are consequences of one root-cause finding (e.g., 64 bundle-audit CVEs all inherited from EOL Rails), each gets demoted by one tier and linked via `severity_inherited_from`. influapp dogfood: 51 demoted, blocker_pct 26.8% → 11.3%.
- **C2 threshold raised** from 25% → 30% based on coba evidence (29.4% legitimate blockers post-secrets-scanner).
- **Complete Tier-2 parser set** — `bin/parse-reek` and `bin/parse-rails-best-practices` join the existing rubocop/brakeman/bundle-audit parsers. All Tier-1 and Tier-2 tools from `tooling.md` now have aggregate-or-finding parsers.
- **Live token accounting contract** — documented how the skill consumes harness-exposed per-call usage to populate `cost.actual_*` when available; null fallback when not.
- **`bin/install-hooks`** — pre-commit integration for `bin/scan-secrets --strict`. Closes the audit-catches-existing / hook-prevents-new loop.
- **`docs/lessons-learned.md`** — 8 build insights from the v0.3-v0.5 milestone work that should outlive any single CHANGELOG entry.

### PRs

| PR | Title |
|---|---|
| #57 | Severity inheritance for EOL-derived CVEs |
| #58 | C2 threshold: 25% → 30% |
| #59 | docs/lessons-learned.md |
| #60 | Complete parser set: parse-reek + parse-rails-best-practices |
| #61 | Live token accounting contract |
| #62 | bin/install-hooks for pre-commit secrets scan |

### Schema additions (all backwards-compatible)

- `findings[].severity_inherited_from` (string, optional, finding_id ref).
- `appendices.code_smells` (from parse-reek).
- `appendices.rails_best_practices` (from parse-rails-best-practices).

v0.2-v0.4 reports validate clean against the v0.5 schema.

### Calibration evidence captured

| Metric | influapp v0.4 | influapp v0.5 | coba v0.4 | coba v0.5 |
|---|---|---|---|---|
| `blocker_pct` | 26.8% | **11.3%** | 29.4% | 29.4% |
| `high_pct` | 62.9% | **42.3%** | 41.2% | 41.2% |
| C2 fires (30% threshold) | n/a | no ✓ | n/a | no ✓ |
| Skill ships? | yes (C7 override) | yes (clean) | no (block) | no (block — concentrated breakage genuinely warrants it) |

### Added (v0.5 milestone PR#6 — bin/install-hooks for pre-commit secrets scan)

- **`bin/install-hooks`** — bash script that wires `bin/scan-secrets --strict` into `.git/hooks/pre-commit`. Modes: install (default), `--uninstall`, `--dry-run`, `--help`.
- The hook block is fenced with begin/end markers so install-hooks can find and remove its own block without touching other hook content. Composable with existing pre-commit setups.
- Bypass for one commit via `git commit --no-verify` (documented in the install message).
- The audit catches existing tracked secrets (Step 1.5 scan); the pre-commit hook prevents new ones — closes the loop.
- Closes #56.

### Added (v0.5 milestone PR#5 — Live token accounting contract)

- New "Live token accounting" section in `dimensions/cost-estimation.md` documenting how the skill populates `cost.actual_input_tokens` / `cost.actual_output_tokens` when the harness exposes per-call usage metadata. Three capture points: Bash tool invocations, agent fan-out, synthesis/render.
- When usage is unavailable, fields stay `null` and `audit.ignore_warnings` notes the gap (current behavior, now documented).
- `capture_actual_tokens: true` in `.claude/rails-audit.yml` default; `--no-token-accounting` CLI override.
- Out of scope (documented): non-Claude harness adapters, billing, per-step breakdown.
- Closes #55.

### Added (v0.5 milestone PR#4 — bin/parse-reek + bin/parse-rails-best-practices)

- **`bin/parse-reek`** — reads reek `-f json` output, aggregates by smell type and top affected files. Aggregate-only (per-smell findings would be noise). Emits the `appendices.code_smells` shape.
- **`bin/parse-rails-best-practices`** — reads rails_best_practices `--format json` output, aggregates by check name and top files. Emits the `appendices.rails_best_practices` shape.
- Schema additions (additive): `appendices.code_smells` and `appendices.rails_best_practices` with `total`, `top_*` arrays.
- Both follow the parse-rubocop pattern: aggregate, never per-finding. The Tier-2 parser set is now complete (parse-rubocop, parse-reek, parse-rails-best-practices). Together with parse-brakeman + parse-bundle-audit, every Tier-1 and Tier-2 tool from `tooling.md` has a parser.
- Closes #53, #54.

### Added (v0.5 milestone PR#3 — docs/lessons-learned.md)

- New `docs/lessons-learned.md` capturing 8 build insights from the v0.3-v0.5 milestone work that should outlive any single CHANGELOG entry: the Ruby gsub/last_match gotcha, the C7 dimension-≤4 heuristic, the 25→30% blocker-pct paradox, why JSON-first matters, the bundle-audit-explosion-plus-inheritance pattern, the v0.3.0-as-plugin-restructure surprise, what was hardest to get right, and what I'd do differently next time.
- Closes #52.

### Changed (v0.5 milestone PR#2 — C2 threshold: 25% → 30%)

- C2 (blocker over-use) threshold raised from `> 0.25` to `> 0.30` in `dimensions/self-check.md`.
- Calibration note added explaining the bump: coba's v0.4 dogfood had 29.4% blockers post-secrets-scanner, all correctly tagged per rubric. Old threshold spuriously blocked. 30% is a realistic ceiling.
- Closes #51.

### Added (v0.5 milestone PR#1 — Severity inheritance for EOL-derived CVEs)

- New schema field: `findings[].severity_inherited_from` (additive, optional, finding_id format). When set, the finding is a consequence of the cited root-cause finding and has been demoted by one tier.
- New dimension file `dimensions/severity-inheritance.md` documents the four detection conditions, the inheritance roster (Rails-bundled + Ruby-adjacent gems), the demotion floor (severity=`low`), and the render rules (nested under root, not separate punch-list entries).
- New SKILL.md Step 4.7 between revalidations and self-check.
- **Dogfood evidence on influapp v0.4 → v0.5**: 51 of 97 findings demoted via inheritance from the Ruby/Rails EOL root. `blocker_pct` dropped from 26.8% → 11.3%; `high_pct` dropped from 62.9% → 42.3%. The remaining 13 bundle-audit findings (out of 64) didn't inherit because their gems aren't on the conservative roster (Stripe, AWS-SDK, etc.) — correct behavior.
- coba v0.5 unchanged: zero EOL findings to inherit from (Rails 7.1, Ruby 3.3.7 are both supported).
- Closes #50.

## [0.4.0] — 2026-05-05

> The v0.3 milestone shipped under this version number — v0.3.0 was already used for the plugin-restructure release on the same day. Ten PRs across calibration + synthesis quality + ergonomics + polish phases, driven by the v0.2 dogfood evidence on influapp + coba.

### Highlights

- **Self-check hardened** from warn-only → block-with-override. C1, C2, C3 now block at Step 5.5 unless C7 overrides or the user explicitly accepts/demotes.
- **Calibration override C7** suppresses C1/C2 when blockers are sed-verified and the project legitimately has broad defect density. Both v0.2 dogfoods would have qualified.
- **Stale-coverage check C6** catches the SimpleCov-data-not-refreshed-in-CI failure mode both v0.2 dogfoods exhibited.
- **Tracked-secrets scanner** (`bin/scan-secrets`) runs as Step 1.5; auto-blocker findings on private keys / credentials / cloud SA JSONs. Found `lib/certs/production.pem` on influapp the v0.2 agents missed.
- **Tool-output parsers** (`bin/parse-brakeman`, `parse-bundle-audit`, `parse-rubocop`) convert raw tool JSON/text into ready-to-merge finding stubs with stable fingerprints. Surfaced 64 individual bundle-audit advisories on influapp where v0.2 had 3 aggregated findings.
- **Security revalidation** (Step 4.6) mirrors v0.2's money revalidation. Six checks: token comparison, IDOR, trusted-header identity, SQL interpolation, open redirect, SSRF.
- **`--continue` / `--from-findings`** re-run modes skip detect + agent fan-out (~5K vs ~50K tokens).
- **`--only-cluster=A,B`** cluster-level scoping atop v0.2's per-dimension scoping.
- **`--track-renames`** opt-in fuzzy matching for moved findings across file renames; preserves first_seen for age tracking.
- **Multi-file output** when reports exceed 30 KB (split into summary / findings / appendix sub-files).
- **Link-rot CI** weekly checks reference URLs in dimension files; opens tracking issue on rot.

### Added (v0.3 milestone PR#10 — Link-rot CI for reference points)

- New GitHub Actions workflow `.github/workflows/check-references.yml`. Runs weekly (Mondays 09:00 UTC) and on manual dispatch. Extracts URLs from the `## Reference points` sections of `skills/rails-audit/dimensions/*.md`, HEAD-checks each, and opens (or updates) a tracking issue tagged `link-rot` listing the rotted URLs.
- Workflow is read-only on the repo content; only writes to issues. Uses `secrets.GITHUB_TOKEN` (no PAT needed).
- New `link-rot` label created on the repo for tagging the auto-generated issue.
- Closes #35.

### Added (v0.3 milestone PR#9 — Multi-file output for large reports)

- **Single-file mode** (default, < 30 KB) — unchanged: `report-YYYY-MM-DD.{json,md}`.
- **Multi-file mode** (≥ 30 KB) — splits into 4 files:
  - `report-YYYY-MM-DD.json` (single source of truth, unchanged)
  - `report-YYYY-MM-DD.md` (top-level index with anchor links)
  - `report-YYYY-MM-DD-summary.md` (exec summary + scorecards + fix sequence)
  - `report-YYYY-MM-DD-findings.md` (full punch list)
  - `report-YYYY-MM-DD-appendix.md` (tooling, coverage, hotspots, ignored, cost)
- Sub-files inherit the report header so each is self-contained when read alone.
- Force a mode via `--single-file` or `--multi-file`; default measures and chooses.
- Threshold (30 KB) tunable via `.claude/rails-audit.yml` if needed in future.
- influapp v0.2 markdown is ~22 KB; coba ~25 KB. Both stay single-file. Trigger fires on larger projects (e.g. 50+ findings) or `--deep` mode reports.
- Closes #36.

### Added (v0.3 milestone PR#8 — Rename detection in fingerprints)

- **`--track-renames`** CLI flag (and `track_renames: true` in `.claude/rails-audit.yml`) opts in to fuzzy-matching fixed+new pairs as moved findings. Default off — keeps trends precise.
- **Algorithm**: pair a `fixed_id` with a `new_id` if same `primary_dimension` + same finding-type + `evidence.normalized` Levenshtein distance ≤ 20% of prior length OR ≤ 30 chars absolute (whichever is larger). Tunable via `rename_distance_pct`.
- **Schema additive**: `trend.moved_ids[]` array with `{prior_id, current_id, prior_location, current_location, match_score}`. Removed pairs are taken out of `fixed_ids` and `new_ids`. v0.2/v0.3 reports validate clean against the v0.4 schema.
- `first_seen` propagates from prior to current for moved findings — age preserved.
- Trade-off documented: false positives possible (two unrelated `update_column` findings could match). False positives visible in the trend table for user inspection.
- Closes #34.

### Added (v0.3 milestone PR#7 — Cluster-level scoping)

- **`--only-cluster=<list>`** and **`--exclude-cluster=<list>`** in SKILL.md "Scope arguments". Accepts cluster letters (`A`/`B`/`C`/`D`) or English aliases (`spec`/`deploy`/`health`/`security`).
- **Cluster aliases** documented in their own section: A=Spec/Coverage, B=Deploy/CI/Obs+Foundation, C=Code Health (9 dimensions), D=Security/Money/Gov+Authz.
- **Note on `security` ambiguity**: in `--only` it means the single dimension; in `--only-cluster` it means the broad cluster D. Documented inline.
- **Mutually-exclusive validation**: at most one of `--only`/`--exclude`/`--only-cluster`/`--exclude-cluster` may be set.
- Closes #33.

### Added (v0.3 milestone PR#6 — Tool-output parsers: bin/parse-brakeman, parse-bundle-audit, parse-rubocop)

- **`bin/parse-brakeman`** — read `tmp/rails-audit/brakeman.json`, emit finding stubs ready to merge into `findings[]`. Maps brakeman warning types to dimensions + severity (e.g. SQL Injection → blocker; Format Validation → high; Cross Site Scripting → high). Confidence levels (High/Medium/Weak) modulate severity. Filters known false positives (array-form `Open3.capture3` — argv, no shell). Stable fingerprints prefixed `f-bk-`.
- **`bin/parse-bundle-audit`** — read `bundle-audit check` text output, parse text-record format (`Name:`, `Version:`, `CVE:`, etc.), emit stubs. Maps `Criticality` to severity (Critical/High → blocker; Medium → high; Unknown → high conservatively). Stable fingerprints prefixed `f-ba-`. Discovered a Ruby gotcha during dogfood: `Regexp.last_match` is clobbered by intervening `String#gsub` calls — fixed by capturing match groups into local variables before subsequent regex operations.
- **`bin/parse-rubocop`** — read `rubocop --format json`, aggregate by severity + cop. Emits the `appendices.rubocop_offenses` shape from `report.schema.json`. Does not produce per-offense findings (style offenses are noise as findings; aggregate is the right granularity).
- **`SKILL.md` Step 2** updated to invoke the parsers after tool runs; their JSON outputs land in `tmp/rails-audit/*-stubs.json` and merge into synthesis at Step 4.
- All three scripts: stdlib only, `--table` for human-readable, `--help` for usage.
- Calibration evidence: parse-bundle-audit on influapp surfaces **64 advisories** (17 blockers + 46 highs) — the v0.2 audit had "uri CVEs" + "thor CVE" as a 3-finding aggregate. Granularity unlocks per-CVE trend tracking.
- Closes #30.

### Added (v0.3 milestone PR#5 — Security-and-authz revalidation pass)

- **`dimensions/security-revalidation.md`** — focused second pass on every finding tagged `security-and-authz` (primary or secondary). Mirrors money-revalidation pattern. Six checks: S-RV-1 token comparison (`secure_compare` discipline), S-RV-2 IDOR scoping, S-RV-3 trusted-header identity (JWT verification), S-RV-4 SQL interpolation (blocker default), S-RV-5 open redirect (host allowlist), S-RV-6 SSRF (private-IP block).
- **`SKILL.md`** new **Step 4.6** between money revalidation (4.5) and self-check (5.5). Sets `findings[<id>].security_revalidation` to `confirmed`/`refined`/`rejected`/`promoted`.
- Schema additive: `findings[].security_revalidation` field with same shape as `money_revalidation`. v0.2/v0.3 reports validate clean.
- Closes #29.

### Changed (v0.3 milestone PR#4 — Harden self-check: warn-only → block-with-override)

- **`dimensions/self-check.md`** — C1 (severity inflation), C2 (blocker over-use), C3 (unverified blocker) now block at Step 5.5 unless overridden. C4, C5, C6 remain warn-only. C7 still suppresses C1/C2 (its whole purpose).
- **`prompts.md` P7** — redesigned. Was `show / demote / accept` with default `show` (v0.2/v0.3). Now `block / demote / accept` with default `block`. New auto-demote algorithm specified (smallest-evidence-first heuristic for C1; small-time-estimate-first for C2). `accept` requires a free-form non-empty reason recorded in `audit.calibration_overrides[]`.
- **`SKILL.md` Step 5.5** — explicit block behavior documented; partial JSON written on block; markdown not rendered.
- **Schema** — new `audit.calibration_overrides[]` field (additive). Records `{check_id, override_reason, override_at, override_by}` when P7 `accept` is picked.
- C3 (unverified blocker) always blocks with no `accept` option — hallucination risk must be addressed.
- Backwards-compat: v0.2/v0.3 reports validate cleanly against the v0.4 schema (the new `calibration_overrides` is optional).
- Closes #26.

### Added (v0.3 milestone PR#3 — Re-run modes: --continue and --from-findings)

- **`--continue`** — load most recent `report-*.json` in `tmp/rails-audit/`; re-run only Step 1.5 (secrets scan) + Steps 4.4 onward (ignores, revalidations, self-check, trend, render). Skips detect + agent fan-out. ~5K tokens vs ~50K for a fresh standard audit.
- **`--from-findings=<path>`** — same shape but loads from an explicit JSON path (hand-edited or copied from another run). Useful for what-if analysis.
- Schema update: `audit.mode` enum extended with `"continue"` and `"from-findings"` (additive, backwards-compatible).
- Behavior documented in `SKILL.md` "Re-run modes" section: skip Steps 1–3, run secrets scan + Steps 4.4 onward, link trend chain via the loaded report's prior_report_path.
- Hard rules: mutually exclusive with each other and with the initial-audit modes; schema_version of loaded report must match current.
- Closes #32.

### Added (v0.3 milestone PR#2 — Tracked-secrets scanner: bin/scan-secrets + Step 1.5)

- **`bin/scan-secrets`** — Ruby script (stdlib only) enumerating `git ls-files` and matching against secret-shaped filename patterns. Emits human-readable table by default; `--json` for machine-readable; `--strict` for CI exit-code mode.
- Patterns include: `.pem`, `.p12`, `.pfx`, `.keystore`, `id_rsa`/`id_ed25519`, `*credentials*.json` (including google-cloud, firebase-admin, gcp-key, aws-credentials), `credentials.txt`/`secrets.txt`, `.env.production`/`.env.*`, `circle_wallets.txt`, PII-shaped CSVs (`users.csv`, `members.csv`, etc.).
- Excludes public certs (`.crt`, `.cer`, `.cert`, `.ca`, `.pub`), `*.example`/`*.sample`/`*.template`, files under `spec/`/`test/`/`fixtures/`.
- **`SKILL.md` Step 1.5** invokes the scanner before static tooling. Findings auto-tagged blocker (or high for ambiguous), `phase: 1`, `primary_dimension: security-and-authz`. Routed straight into `findings[]` during synthesis.
- Calibration evidence: scanner found a tracked `.pem` in influapp that the v0.2 agent fan-out missed. Coba scan finds 8 tracked secrets (matching the v0.2 agent's manual finding plus 2 more).
- Closes #31.

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

[Unreleased]: https://github.com/kurenn/rails-audit/compare/v0.5.2...HEAD
[0.5.2]: https://github.com/kurenn/rails-audit/releases/tag/v0.5.2
[0.5.1]: https://github.com/kurenn/rails-audit/releases/tag/v0.5.1
[0.5.0]: https://github.com/kurenn/rails-audit/releases/tag/v0.5.0
[0.4.0]: https://github.com/kurenn/rails-audit/releases/tag/v0.4.0
[0.3.0]: https://github.com/kurenn/rails-audit/releases/tag/v0.3.0
[0.2.0]: https://github.com/kurenn/rails-audit/releases/tag/v0.2.0
[0.1.0]: https://github.com/kurenn/rails-audit/releases/tag/v0.1.0
