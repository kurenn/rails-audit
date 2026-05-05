---
name: rails-audit
description: Comprehensive Rails project stability audit across 18 dimensions — foundation, domain shape, specs, coverage, deploy/CI, security, authorization, money paths, code health, code smells, performance, reliability, observability, jobs, data integrity, governance, DX, and cost. Use when user asks to "audit this Rails project", "stability assessment", "code health check", "pre-launch review", "Rails security audit", or "is this ready to ship". Produces a severity-ranked markdown report with a recommended fix sequence. Read-only — never edits code.
---

# Rails Audit

Run a structured stability audit of a Ruby on Rails codebase and produce a single markdown report with a severity-ranked punch list and a recommended fix sequence.

## Modes

- `--quick` — static-only checks (bundle audit, brakeman, rubocop, basic greps). Single agent. ~3 min. Suitable for PR-time.
- `--standard` (default) — full audit with parallel subagents. ~10–15 min.
- `--deep` — standard + boots app, runs subset of specs, mutation testing on critical paths. ~30+ min.

If the user invokes `/rails-audit` with no mode, ask using prompt **P1** in `prompts.md` — then proceed.

## Scope arguments

By default an audit covers all 18 dimensions. Two args narrow the scope:

- **`--only=<comma-list>`** — run only these dimensions (or aliases). Example: `/rails-audit --only=money,security`.
- **`--exclude=<comma-list>`** — run all dimensions *except* these. Example: `/rails-audit --exclude=dx-and-cost,observability`.
- **Positional shortcut** — a single non-flag arg is treated as `--only=<arg>`. Example: `/rails-audit money` ≡ `/rails-audit --only=money`.

Mutually exclusive: `--only` and `--exclude` cannot be combined; reject with an error.

### Aliases (case-insensitive)

| Alias | Resolves to |
|---|---|
| `all` | every dimension |
| `money` | `money-and-payments` |
| `security` | `security-and-authz` |
| `auth` | `authorization` |
| `authz` | `authorization` |
| `deploy` | `deploy-and-ci` |
| `ci` | `deploy-and-ci` |
| `specs` | `spec-stability` |
| `coverage` | `test-coverage` |
| `code-smells` | `code-smells` |
| `code` | `code-smells` + `risk-hotspots` |
| `perf` | `performance` + `reliability` |
| `jobs` | `background-jobs` |
| `obs` | `observability` |
| `data` | `data-integrity` + `data-governance` |
| `dx` | `developer-experience` |
| `cost` | `cost-and-scaling` |

Dimension names accepted directly (e.g. `money-and-payments`) as well.

### Validation

- Unknown dimension or alias → reject with `"Unknown dimension '<name>'. Did you mean '<closest-match>'?"` and exit. Use Levenshtein distance ≤2 for the suggestion.
- `--only=` empty (no dimensions) → reject.
- Both `--only` and `--exclude` set → reject.

### Behavior

When scope is narrower than `all`:

1. **Step 3 fan-out** skips clusters whose dimensions are entirely outside the scope. Cluster ↔ dimension mapping:
   - **A** Spec & Coverage: `spec-stability`, `test-coverage`
   - **B** Deploy & CI: `foundation`, `deploy-and-ci`, `observability`
   - **C** Code Health: `domain-shape`, `risk-hotspots`, `code-smells`, `performance`, `reliability`, `background-jobs`, `data-integrity`, `developer-experience`, `cost-and-scaling`
   - **D** Security & Money: `security-and-authz`, `authorization`, `money-and-payments`, `data-governance`
   A cluster that has at least one in-scope dimension still runs; agents are told to focus on the in-scope subset.

2. **`audit.scope[]` in the report** lists the resolved dimension names (not aliases). For full audits: `["all"]`. For scoped: `["money-and-payments", "security-and-authz", ...]`.

3. **Scorecards and findings** are filtered to in-scope dimensions only. Findings whose `primary_dimension` is in scope are included; those whose only in-scope tag is in `secondary_dimensions[]` are also included (tagging is the point of cross-cuts).

4. **Renumbering**. The dimension index in the markdown scorecard table re-numbers from 1 (don't render `#3 #5 #7` — render `#1 #2 #3`).

5. **Money-revalidation (Step 4.5)** runs only if `money-and-payments` is in scope.

6. **Self-check (Step 5.5)** runs in scoped mode too — calibration percentages compute over the scoped finding set.

7. **Trend (Step 7)** compares only the scoped slice of findings against the prior report's matching slice.

## Workflow

### Step 0. Provision tooling

The audit's value is bounded by which static tools are available. **Before fanning out**, ensure the project has the tools the skill expects.

1. Detect what's installed (see `tooling.md` for the contract).

2. **If any Tier-1 (required) tool is missing**, ask per missing tool using prompt **P2** in `prompts.md`. The prompt's three answers (`gemfile` / `global` / `skip`) each have well-defined behavior and a fallback.

3. **For Tier-2 (recommended) tools**, ask once using prompt **P3** in `prompts.md`.

4. **Tier-3 (optional / deep-mode) tools** are *only* prompted in `--deep` mode, and are always opt-in.

5. **Hard floor**: if the project declines all of `{rubocop, brakeman, simplecov}`, abort the audit (P2's hard-floor branch handles this).

6. **Don't auto-add** anything without explicit consent. The Gemfile is the user's source of truth — modifying it silently is out of scope.

7. **Cache provisioning state**: write `tmp/rails-audit/tools-detected.json` so subsequent runs can fast-path. Re-detect on `--quick` only if the cache is older than 7 days.

8. **First-run profile init**: if `.claude/rails-audit.yml` doesn't exist and the skill auto-detected stack values, ask using prompt **P4** in `prompts.md`. Skip this step on subsequent runs.

A reusable Gemfile snippet for the user to paste manually if they prefer is documented in `tooling.md` under "Recommended Gemfile additions".

### Step 0.5. Estimate token cost

After provisioning (Step 0) but before detect/fan-out, compute a token budget estimate using the heuristic in `dimensions/cost-estimation.md`. Store in `cost.estimated_input_tokens` and `cost.estimated_output_tokens`.

If `--budget=<N>` is set explicitly, store in `cost.budget_tokens`.

If `--budget` is **not** set AND `estimated_input_tokens > 30_000`, run prompt **P5** in `prompts.md`. Accepted answers:

- `Y` (default) — proceed with no cap.
- `n` — abort.
- `budget=N` (or a bare number) — set `cost.budget_tokens = N` and proceed.

When a budget is in effect, gate every agent call: before launching, compute `remaining = budget - usage_so_far`. If the agent's conservative upper-bound cost (use `per_dim_in + per_dim_out` per dimension in scope) would exceed `remaining`, skip the call. When the budget is hit, save the partial JSON, set `summary.verdict` to note the abort, and render a partial markdown report flagging missing dimensions.

Hard floor: `budget_tokens < 5000` is rejected (no audit can fit).

### Step 1. Detect

Confirm Rails project, capture orientation:

```bash
test -f Gemfile && grep -q "gem ['\"]rails['\"]" Gemfile || echo "NOT_RAILS"
test -f config/application.rb || echo "NOT_RAILS"
ruby -e 'puts RUBY_VERSION' 2>/dev/null
grep -E "^ruby|gem ['\"]rails['\"]" Gemfile
ls app/ config/ .github/workflows/ 2>/dev/null
test -f Procfile -o -f Dockerfile -o -f fly.toml -o -f config/deploy.yml && echo "deploy artifacts found"
```

Read `.claude/rails-audit.yml` if present (project profile — see "Project profile" below). Otherwise auto-detect:
- **Deploy target**: workflow filenames (`deploy-to-cloud-run.yml` → Cloud Run; `kamal*` → Kamal; presence of `Procfile` + Heroku buildpacks → Heroku; `fly.toml` → Fly).
- **Job adapter**: `Gemfile` greps — `sidekiq`, `cloudtasker`, `good_job`, `delayed_job`, `resque`.
- **Auth**: `Gemfile` greps — `devise`, `warden`, `jwt`, `clearance`, `doorkeeper`.
- **Test framework**: `.rspec` exists → RSpec; `test/` exists → Minitest.

### Step 1.5. Scan tracked secrets (added v0.3)

Before running static tooling, run `bin/scan-secrets --json > tmp/rails-audit/secrets.json` from the project root. The script enumerates `git ls-files`, matches against secret-shaped filename patterns (`.pem`, `.p12`, `.pfx`, `.keystore`, `id_rsa`, `*credentials*.json`, `credentials.txt`, `secrets.txt`, `.env.production`, etc.), and emits ready-to-merge finding stubs.

Each match is auto-blocker by default (private keys, GCP service-account JSONs, AWS credentials) — tracked secrets are permanently compromised by definition. Exclusions: `.crt`/`.cer`/`.cert`/`.ca`/`.pub` (public certs), files matching `*.example`, `*.sample`, `*.template`, anything under `spec/`, `test/`, or `fixtures/`.

The findings stubs go directly into `findings[]` during synthesis (Step 4). Their `phase` is always 1 — secret rotation must precede any other work.

If a finding is a false positive (e.g. an intentionally-tracked dummy `.pem` for testing), the user can add it to `.audit-ignore.yml` post-audit. The scanner does not attempt to read file *contents* — only filename patterns — so it cannot distinguish "dummy test cert" from "production key" by content. The `.audit-ignore.yml` mechanism is the right escape hatch.

The scanner is also available standalone:

```bash
bin/scan-secrets                # human-readable table
bin/scan-secrets --strict       # exit 1 if any tracked secret-shaped file is found (CI gate)
```

### Step 2. Run static tooling in parallel

See `tooling.md` for the full contract. Capture all output to `tmp/rails-audit/`:

```bash
mkdir -p tmp/rails-audit
```

Then run available tools in parallel Bash calls. Skip silently if not installed (note absence in the report's tooling section). For Bundler-managed tools, prefix with `bundle exec` if Gemfile lists them; otherwise try the bare command.

### Step 3. Fan out to subagents

Launch up to **4 Explore agents IN PARALLEL** (single message, multiple Agent calls). When `audit.scope` is narrower than `all`, skip clusters whose dimensions are entirely outside the scope (see "Scope arguments" → cluster mapping). Clusters that have at least one in-scope dimension still run, but the agent brief instructs them to focus on the in-scope subset only.

Each agent gets:
- The relevant dimension file path(s) from `dimensions/`
- The severity rubric path: `rubric.md`
- The static-tool output paths from step 2
- Instructions to return a structured punch list, ≤800 words

**Cluster A — Spec & Coverage** (`dimensions/spec-and-coverage.md`)
Audit the test suite for stability and the coverage signals that matter.

**Cluster B — Deploy & CI** (`dimensions/deploy-ci.md`, `dimensions/observability.md`)
Audit deploy pipeline, prod config, secrets, health checks, rollback, and observability.

**Cluster C — Code Health** (`dimensions/domain-shape.md`, `dimensions/code-health.md`, `dimensions/performance-reliability.md`, `dimensions/background-jobs.md`)
Audit risk hotspots, smells, antipatterns, performance, jobs, data integrity.

**Cluster D — Security & Money** (`dimensions/security-and-authz.md`, `dimensions/money-and-payments.md`, `dimensions/data-governance.md`)
Audit AuthN/AuthZ, OWASP-top patterns, payment/idempotency, PII handling.

For each agent, write a self-contained brief: what to investigate, where to look, what to return. Don't say "based on the dimension file"; quote the specific checks the agent should run.

### Step 4. Synthesize → JSON

Read the four agent outputs and produce **a JSON object conforming to `schema/report.schema.json`**. JSON is the source of truth; the markdown report is rendered from it (Step 5). This step is the one that must get the contract right.

1. **Verify loud claims**. Before writing "force_ssl is disabled" or "token comparison is timing-vulnerable" into a finding, read the cited line yourself with `sed -n '<line>p' <file>`. Agents hallucinate line numbers. Findings whose cite couldn't be verified get `self_check.status: "unverified"` and surface in the self-check section (see PR#5).
2. **Deduplicate and tag dimensions**. Same finding often surfaces in multiple clusters. Merge into one `findings[]` entry. Pick the most-load-bearing dimension as `primary_dimension`; tag all other applicable dimensions in `secondary_dimensions[]`. Each dimension file documents its known cross-cuts (`## Cross-cuts` section) — read those before tagging. Examples: `update_column(:stripe_customer_id)` is **code-smells** primary with **money-and-payments** + **data-integrity** secondary; a Stripe webhook missing event-ID dedup is **money-and-payments** primary with **security-and-authz** secondary; `force_ssl` disabled is **deploy-and-ci** primary with **security-and-authz** secondary. Per-dimension scorecards count findings where the dimension appears in `primary_dimension` OR any `secondary_dimensions[]` — so accurate tagging makes the scorecards trustworthy.
3. **Compute fingerprints**. For each finding, `id = "f-" + first 16 hex chars of SHA256(primary_dimension + file_path + finding_type + normalize(evidence_snippet))`. The `normalize` function: strip leading/trailing whitespace, remove inline + block comments, collapse internal whitespace runs to single space, preserve case. Store the normalized snippet in `evidence.normalized` so the fingerprint is reproducible.
4. **Apply rubric**. Severity from `rubric.md`. Don't inflate to "high" to seem useful. Money/auth defects default to blocker unless verified non-exploitable.
5. **Assign each finding to a phase** (`phase: 1..5`) — phase numbers come from your fix-sequence ordering. Phases are deduped + ordered so each unblocks the next.
6. **Populate `summary`, `scorecards`, `tooling`, `fix_sequence`** as the schema requires.

The output of this step is **a single JSON object**. Validate it against `schema/report.schema.json` before proceeding.

### Step 4.4. Apply `.audit-ignore.yml`

Read `.audit-ignore.yml` from the repo root if present. Each entry is a fingerprint-keyed acknowledgement (see `examples/.audit-ignore.yml.example`). Required fields per entry: `id` (matches `findings[].id`) and `reason` (non-empty). Optional: `acknowledged_by`, `expires_at`.

For each entry:

1. **If `expires_at` is present and < today** — the ignore is expired. The matching finding stays in the punch list with a `_(ignore expired YYYY-MM-DD)_` note appended to its explanation. Do **not** add to `ignored_findings[]`.
2. **If the entry's `id` matches no current finding** — push a string to `audit.ignore_warnings[]` describing the stale ignore (so it can be cleaned up). Skip.
3. **Otherwise** — set `findings[<id>].ignored = true` and add an entry to top-level `ignored_findings[]` with `id`, `reason`, `acknowledged_by`, `expires_at`, `expired: false`.

Findings with `ignored: true` are excluded from:
- The punch list rendered in markdown (they go to Appendix D instead)
- Step 4.5 money-path re-validation (no point re-checking suppressed findings)
- Step 5.5 self-check calibration percentages (suppressed findings don't inflate the high/blocker pcts)

But they **are** included in:
- Trend tracking (Step 7) — an ignored finding still counts as "persisted" if it was in the prior report
- The fingerprint corpus for future runs (so the ignore can match stably)

If the user wants to **add** a new ignore entry interactively during an audit, use prompt **P6** in `prompts.md` (free-form reason; default 90-day expiry).

The schema and template already support `ignored_findings[]` and `audit.ignore_warnings[]` from PR#1. No schema or template change needed.

### Step 4.5. Money-path re-validation

Before self-check, run a focused second pass on every finding tagged `money-and-payments` in `primary_dimension` OR `secondary_dimensions[]`. See `dimensions/money-revalidation.md` for the full checklist.

For each finding in scope:

1. Re-read the cited file region with `sed -n '<start>,<end>p' <file>`.
2. Apply the money-specific checks from `dimensions/money-and-payments.md`: idempotency-key derivation (M-RV-1), transaction-boundary ordering (M-RV-2), webhook event-ID dedup (M-RV-3), money column types (M-RV-4), refund idempotency (M-RV-5), audit trail completeness (M-RV-6).
3. Set `findings[<id>].money_revalidation` to one of:
   - `confirmed` — finding stands as written
   - `refined` — same finding, evidence/explanation/fix_sketch improved with money-specific detail (record `notes`)
   - `rejected` — false positive on closer look (set `ignored: true`, record `notes`)
   - `promoted` — severity bumped one tier because a money-specific check applies (record `notes`)
4. Run the **proactive sweep** on `app/services/payments/*`, `app/controllers/webhooks/stripe_*`, `app/jobs/*payment*`, `app/jobs/*payout*`, and money-shaped columns in `db/schema.rb` to surface findings that synthesis may have missed. New findings get `tool_origin: "money-revalidation"`.

Re-validation never modifies `id`, `primary_dimension`, or `secondary_dimensions[]`. Schema field `findings[].money_revalidation` is already defined as nullable in `schema/report.schema.json` from PR#1; the template renders `_Re-validated: <status>_` next to fix sketches.

### Step 5.5 (was 5). Self-check the report

Before rendering, run the self-check defined in `dimensions/self-check.md`. The five checks (C1–C5) operate on the JSON from Step 4:

- **C1** Severity inflation — warn if `>40%` of findings are `high`
- **C2** Blocker over-use — warn if `>25%` are `blocker`
- **C3** Unverified blocker — for each blocker, `sed -n '<line>p' <file>` must contain a substring of `evidence.normalized`. Fail → mark `findings[<id>].self_check.status = "unverified"` and add to `self_check.unverified_blockers[]`
- **C4** Phase dependency mismatch — `findings[id].phase` must equal `fix_sequence[].finding_ids` references
- **C5** Scorecard ↔ finding-count mismatch — per-dimension score must fall in the expected band given the weighted finding count for that dimension (blocker=4, high=3, medium=2, low=1)

In **v0.2** these are warn-only: results land in `self_check.calibration.warnings[]` and per-finding `self_check{}`, no findings are removed or demoted. If warnings fire and the user is interactively driving the audit, prompt **P7** (`prompts.md`) — show / demote / accept. Default is `show`. v0.2 honors `accept` automatically (skill ships the report); `demote` is a no-op in v0.2 and lands as a real action in v0.3.

Self-check meta-findings render in the markdown report as a `## Self-check` section above the punch list (template already supports this).

### Step 5.7. Compute trend

If a prior `report-*.json` exists in `tmp/rails-audit/`, compute the per-finding diff per `dimensions/trend-tracking.md` and populate `trend{}` in the JSON before rendering.

Algorithm:

1. List `tmp/rails-audit/report-*.json` and pick the most recent file whose date is older than today.
2. If none found, set `trend = null` and skip.
3. If found but `schema_version` ≠ current, set `trend = null` and push to `audit.ignore_warnings[]` (`"Prior report at <path> has incompatible schema_version <N>; trend skipped."`).
4. Otherwise compute `fixed_ids` / `new_ids` / `persisted_ids` by fingerprint set diff. Filter by `audit.scope` if the current run is scoped.
5. Propagate `first_seen`: for persisted findings, copy from prior; for new findings, set to today.
6. Mark persisted findings ≥ 30 days old as `stale` for the render layer (no schema field — it's a derived render flag).

### Step 6 (was 5). Render markdown

Apply `output-template.md` to the JSON from Step 4 (now possibly mutated by Step 5.5). The template is authoritative — placeholder names refer to JSON paths. Filters (`date`, `dimension_label`, `range_str`, `percent`, `join`, `where`, `sort_by`) are deterministic; same JSON in produces same markdown out.

### Step 7 (was 6). Write outputs

Save **both** files to `tmp/rails-audit/`:

- `report-YYYY-MM-DD.json` — the structured source of truth (use today's actual date)
- `report-YYYY-MM-DD.md`   — the rendered view

If a prior `report-*.json` exists in the same directory, populate `trend{}` in the new JSON before rendering (see PR#10 — finding-level diff by fingerprint).

### Step 8 (was 7). Brief the user

≤200 words back to the user: top 3 blockers, recommended next action, links to both the `.json` and `.md` files.

## Critical rules

- **Always invoke external tools, never re-implement them.** The skill's value is synthesis (deduping, ranking, sequencing), not detection. `brakeman`, `bundle-audit`, `rubocop`, `reek`, `rails_best_practices`, `simplecov` already do detection well.
- **Cite file paths with line numbers.** `app/controllers/api/authenticated_controller.rb:33` — not "in the auth controller".
- **Verify loud claims.** Read the cited file before writing the finding. Especially for security/money claims.
- **Severity must follow the rubric.** Money/auth defects are blocker by default.
- **Read-only.** Never edit code during an audit. Offer to fix afterward if the user asks.
- **Don't pad.** A short, accurate report with 8 real findings beats a long one with 30 findings half of which are noise.
- **Adapt to project profile.** A Cloud Run project gets different deploy checks than a Kamal one. A Sidekiq project gets different job checks than a Cloudtasker one.

## Project profile

Optional `.claude/rails-audit.yml` at repo root. When missing, auto-detect from Gemfile + workflows + paths.

```yaml
deploy_target: cloud_run         # cloud_run | heroku | kamal | render | ecs | fly | other
job_adapter: cloudtasker         # sidekiq | resque | good_job | cloudtasker | delayed_job | other
auth_strategy: warden_jwt        # devise | warden | jwt | clearance | doorkeeper | custom
money_columns:                   # checked for Decimal type
  - transactions.amount_cents
  - payouts.amount
critical_paths:                  # paths held to higher coverage + security bar
  - app/services/payments/
  - app/controllers/webhooks/
  - app/services/identity_platform/
ignore_paths:                    # excluded from smell/coverage checks
  - app/admin/
  - lib/legacy/
```

## Files

- `rubric.md` — severity definitions and calibration examples
- `tooling.md` — tool contract, detection, and invocation patterns
- `prompts.md` — every user-facing question (P1–P7) with answers, defaults, and fallbacks
- `schema/report.schema.json` — **JSON Schema (draft 2020-12) for the report** — the contract; everything else is a view
- `output-template.md` — markdown render template applied to the JSON
- `dimensions/` — per-dimension check lists, one file per cluster
- `examples/sample-report.json` — example structured output
- `examples/sample-report.md` — example rendered output (derived from `sample-report.json`)

When working on a specific dimension, read the corresponding `dimensions/*.md` file. Don't try to keep all 18 dimensions in head at once.
