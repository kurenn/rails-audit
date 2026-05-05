# Report rendering template

JSON is the source of truth (`schema/report.schema.json`). This file defines how to render that JSON to markdown for human consumption.

The renderer is Claude itself, applied during the skill's render step (Step 5 of `SKILL.md`). Treat the template below as authoritative — placeholder names refer to JSON paths.

## Conventions

- **`{{path.to.field}}`** — single value substitution
- **`{{#each items}}…{{/each}}`** — iterate over array
- **`{{#if condition}}…{{/if}}`** — conditional block
- **`{{path | filter}}`** — apply transformation (date, dimension_label, range_str, percent, join, slice, sort_by)
- Filters must be deterministic — same JSON in → same markdown out

## Filter reference

| Filter | Input | Output |
|---|---|---|
| `date` | ISO 8601 datetime | `YYYY-MM-DD` |
| `dimension_label` | dimension enum value (e.g. `money-and-payments`) | Title Case (e.g. `Money & Payments`) |
| `range_str` | `[start, end]` array | `start-end`, or just `start` if equal |
| `percent` | float 0–1 | `XX%` |
| `join " · "` | array | string joined by separator |
| `slice 0 5` | array | first 5 elements |
| `sort_by severity` | findings array | sorted blocker → low |
| `where severity=="X"` | findings array | filtered |
| `count_where ...` | array | integer count |

## Empty states

- An empty section is omitted entirely — never render `## Section\n\n(none)\n`.
- `top_blocker_ids` empty ⇒ skip the "Top 3 blockers" line; lead with verdict only.
- `findings` empty per severity tier ⇒ omit the entire `### Tier` heading.
- `self_check.calibration.warnings` empty ⇒ omit the `## Self-check` section.
- `trend` absent ⇒ omit the `## Trend` section (don't render "no prior report").

---

## Markdown skeleton

````markdown
# Rails Stability Audit — {{audit.project_name}}

**Date:** {{audit.started_at | date}}
**Audited commit:** `{{audit.commit}}` on `{{audit.branch}}`
**Mode:** {{audit.mode}}{{#if audit.scope[0] != "all"}} · **Scope:** {{audit.scope | dimension_label | join ", "}}{{/if}}
**Stack detected:** Rails {{audit.stack.rails}} / Ruby {{audit.stack.ruby}} / {{audit.stack.deploy_target}} / {{audit.stack.job_adapter}} / {{audit.stack.auth_strategy}}{{#if audit.stack.test_framework}} / {{audit.stack.test_framework}}{{/if}}

---

## Executive summary

**Overall risk: {{summary.risk_score}}/10** — {{summary.verdict}}

{{#if summary.top_blocker_ids}}
**Top {{summary.top_blocker_ids | count}} blockers**:
{{#each summary.top_blocker_ids}}
{{@index_one}}. {{find findings id=@this | title}} — `{{find findings id=@this | location.file}}:{{find findings id=@this | location.line_range | range_str}}`
{{/each}}
{{/if}}

**Recommended next action:** {{summary.next_action}}

---

{{#if self_check.calibration.warnings}}
## Self-check

The skill's calibration check raised {{self_check.calibration.warnings | count}} warnings:

{{#each self_check.calibration.warnings}}
- {{this}}
{{/each}}

{{#if self_check.unverified_blockers}}
**Unverified blockers** ({{self_check.unverified_blockers | count}}): the cited line numbers could not be confirmed by re-reading the file. Treat with skepticism: {{self_check.unverified_blockers | join ", "}}
{{/if}}

---
{{/if}}

## Per-dimension scorecards

| # | Dimension | Score | What would make it a 10 |
|---|---|---|---|
{{#each scorecards}}
| {{@index_one}} | {{dimension | dimension_label}} | {{score}}/10 | {{would_be_10}} |
{{/each}}

Score guide: 0–3 unsafe / 4–6 risky / 7–8 acceptable / 9–10 strong.

---

## Punch list

{{#if findings | where severity=="blocker"}}
### Blocker

{{#each findings | where severity=="blocker" | sort_by phase}}
#### B{{@index_one}}. {{title}} — {{primary_dimension | dimension_label}}{{#if secondary_dimensions}} · {{secondary_dimensions | dimension_label | join " · "}}{{/if}}
**Location:** `{{location.file}}{{#if location.line_range}}:{{location.line_range | range_str}}{{else if location.line}}:{{location.line}}{{/if}}`
**Evidence:**
```
{{evidence.snippet}}
```
**Why this is a blocker:** {{explanation}}
**Fix sketch:** {{fix_sketch}}{{#if time_estimate}} _(≈{{time_estimate}})_{{/if}}{{#if money_revalidation.status}} · _Re-validated: {{money_revalidation.status}}_{{/if}}

{{/each}}
{{/if}}

{{#if findings | where severity=="high"}}
### High

{{#each findings | where severity=="high" | sort_by phase}}
#### H{{@index_one}}. {{title}} — {{primary_dimension | dimension_label}}{{#if secondary_dimensions}} · {{secondary_dimensions | dimension_label | join " · "}}{{/if}}
**Location:** `{{location.file}}{{#if location.line_range}}:{{location.line_range | range_str}}{{else if location.line}}:{{location.line}}{{/if}}`
{{#if evidence.snippet}}**Evidence:**
```
{{evidence.snippet}}
```
{{/if}}
{{explanation}}
**Fix:** {{fix_sketch}}{{#if time_estimate}} _(≈{{time_estimate}})_{{/if}}

{{/each}}
{{/if}}

{{#if findings | where severity=="medium"}}
### Medium

| ID | Finding | Location | Dimension(s) |
|---|---|---|---|
{{#each findings | where severity=="medium" | sort_by phase}}
| M{{@index_one}} | {{title}} | `{{location.file}}{{#if location.line}}:{{location.line}}{{/if}}` | {{primary_dimension | dimension_label}}{{#if secondary_dimensions}} · {{secondary_dimensions | dimension_label | join " · "}}{{/if}} |
{{/each}}
{{/if}}

{{#if findings | where severity=="low" || appendices.rubocop_offenses || appendices.todo_density}}
### Low

| Area | Count | Examples |
|---|---|---|
{{#if appendices.rubocop_offenses.total}}
| RuboCop offenses (convention) | {{appendices.rubocop_offenses.total}} | {{appendices.rubocop_offenses.top_categories | slice 0 3 | join ", "}} |
{{/if}}
{{#if appendices.todo_density.total}}
| TODO/FIXME comments | {{appendices.todo_density.total}} | {{appendices.todo_density.top_files | slice 0 3 | each "{{file}}:{{count}}" | join ", "}} |
{{/if}}
{{#each findings | where severity=="low"}}
| {{primary_dimension | dimension_label}} | 1 | {{title}} — `{{location.file}}` |
{{/each}}
{{/if}}

---

## Recommended fix sequence

{{#each fix_sequence}}
### Phase {{phase}} — {{name}} ({{estimated_effort}})
{{#each finding_ids}}
- [ ] {{find ../findings id=@this | severity_letter}}{{find ../findings id=@this | severity_index}}: {{find ../findings id=@this | title}} (`{{find ../findings id=@this | location.file}}`)
{{/each}}
{{/each}}

**Rationale:** {{fix_sequence | last | rationale}}{{#if fix_sequence | length > 1}}{{!--
  Per-phase rationales:
--}}{{#each fix_sequence}}{{#if rationale}}
- _Phase {{phase}}_: {{rationale}}{{/if}}{{/each}}{{/if}}

---

{{#if trend}}
## Trend (vs {{trend.prior_report_date}})

| Bucket | Count |
|---|---|
| Fixed since prior | {{trend.fixed_ids | count}} |
| New | {{trend.new_ids | count}} |
| Persisted | {{trend.persisted_ids | count}} |

{{#if trend.fixed_ids}}
**Fixed**: {{trend.fixed_ids | each_resolve_title | join ", "}}
{{/if}}
{{#if trend.new_ids}}
**New**: {{trend.new_ids | each_resolve_title | join ", "}}
{{/if}}

---
{{/if}}

## Appendix A — Tooling

| Tool | Status | Version | Notes |
|---|---|---|---|
{{#each tooling.ran}}
| {{name}} | ran | {{version}} | {{#if findings_count}}{{findings_count}} finding(s){{/if}} |
{{/each}}
{{#each tooling.missing}}
| {{name}} | missing | — | {{reason}} |
{{/each}}
{{#each tooling.skipped}}
| {{name}} | skipped | — | {{reason}} |
{{/each}}

{{#if appendices.coverage_map}}
## Appendix B — Coverage map

| Path | Files | With spec | Line cov | Branch cov | Risk | Notes |
|---|---|---|---|---|---|---|
{{#each appendices.coverage_map}}
| `{{path}}` | {{files}} | {{with_spec}} | {{line_cov | percent}} | {{branch_cov | percent}} | {{risk_weight}} | {{notes}} |
{{/each}}
{{/if}}

{{#if appendices.risk_hotspots}}
## Appendix C — Top risk hotspots by file size

| File | LOC | Layer | Has spec? |
|---|---|---|---|
{{#each appendices.risk_hotspots}}
| `{{file}}` | {{loc}} | {{layer}} | {{has_spec}} |
{{/each}}
{{/if}}

{{#if ignored_findings}}
## Appendix D — Ignored findings

| ID | Reason | Acknowledged by | Expires |
|---|---|---|---|
{{#each ignored_findings}}
| `{{id}}` | {{reason}} | {{acknowledged_by}} | {{expires_at}}{{#if expired}} ⚠️ EXPIRED{{/if}} |
{{/each}}
{{/if}}

{{#if cost.estimated_input_tokens || cost.actual_input_tokens}}
## Appendix E — Cost recap

|        | Input tokens | Output tokens |
|---|---|---|
| Estimated | {{cost.estimated_input_tokens}} | {{cost.estimated_output_tokens}} |
| Actual    | {{cost.actual_input_tokens}}    | {{cost.actual_output_tokens}} |
{{#if cost.budget_tokens}}| Budget    | {{cost.budget_tokens}} | — |{{/if}}
{{/if}}

---

*Generated by `/rails-audit` v{{skill_version}} — {{audit.mode}} mode*
````

## Rendering rules

These rules are **not negotiable** — they encode the audit-quality bar.

- **No emojis.** Audit reports go to people deciding what to ship; tone is utilitarian.
- **No fluff.** "Excellent work!" / "Looking great!" — cut. State the score, list the gaps.
- **Verbatim evidence.** `evidence.snippet` is copy-pasted from the file, not paraphrased. The renderer does not modify snippets.
- **Cite line numbers.** Every finding entry shows `<file>:<line>` or `<file>:<start>-<end>`.
- **Fix sketches stay short.** Schema enforces `maxLength: 600`; renderer doesn't expand them.
- **Phase sizing must be honest.** `estimated_effort` is a coarse bucket — don't render arbitrary "1.5 days".
- **Scope-aware rendering.** When `audit.scope` excludes dimensions, scorecards and findings for excluded dimensions are not rendered. The dimension index numbering compresses (don't render `#3 #5 #7` — re-number to `#1 #2 #3`).

## What goes where (anti-confusion guide)

| Content | Schema field | Render location |
|---|---|---|
| One-line dimension verdict | `scorecards[].would_be_10` | Scorecard table |
| Per-finding details | `findings[]` | Punch list |
| Suppressed findings | `ignored_findings[]` | Appendix D only — never punch list |
| Tool offense counts (RuboCop, etc.) | `appendices.rubocop_offenses` | Low tier table |
| Coverage map | `appendices.coverage_map` | Appendix B |
| File-size hotspots | `appendices.risk_hotspots` | Appendix C |
| Self-check warnings | `self_check.calibration.warnings` | Above punch list |
| Trend diff | `trend` | After fix sequence |

## Changelog

- **v0.2** (this PR) — JSON-first; renderer template defined here.
- **v0.1** — Free-form prose template (deprecated).
