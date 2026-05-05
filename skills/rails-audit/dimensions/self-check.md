# Self-check (audit-the-audit)

The skill polices its own report before delivering it. Self-check produces *meta-findings* about the audit itself — not the codebase. Outcomes land in JSON `self_check{}` and render in the markdown report as a `## Self-check` section above the punch list.

**v0.4** hardens self-check from warn-only to **block-with-override**: when C1 or C2 fires AND C7 does *not* override, the skill stops at Step 5.5 and runs prompt **P7** (`prompts.md`). The user must pick `block` (default — don't ship until findings are revisited), `demote` (auto-demote inflated severities one tier and proceed), or `accept` (record an override reason in `audit.calibration_overrides[]` and proceed).

C3 (unverified blocker) **always blocks** — no override. An unverified blocker is a hallucination risk; the user has to either re-cite it or remove it.

C4 (phase mismatch), C5 (scorecard mismatch), C6 (stale coverage) remain warn-only — they're informational, not exploit-shaped.

In v0.2 self-check was warn-only across the board. v0.4's hardening reflects calibration evidence from the influapp + coba dogfoods (2026-05-05): both projects' C1/C2 firings turned out to be either real defect density (C7 override applied) OR genuine synthesis inflation (worth blocking). Warn-only let synthesis ship inflated reports; block-with-override forces the conversation.

---

## Run order

Self-check runs as **Step 5.5** in `SKILL.md` — *after* synthesis (Step 4) and the various revalidation passes (4.4–4.6), *before* trend (5.7) and markdown render (6). Self-check may mutate `self_check{}` and individual `findings[].self_check{}` fields. **In v0.4+, self-check may also mutate finding `severity`** when the user picks `demote` at prompt P7 — the demoted finding gets `self_check.status: "demoted"` and a `notes` line explaining which check caused it.

---

## Checks

Each check has: trigger, threshold, output to `self_check.calibration.warnings[]`, side-effect on the JSON.

### C1 — Severity inflation

**Trigger.** Compute `high_pct = (count where severity == "high") / total_findings`.
**Threshold.** Warn if `high_pct > 0.40`.
**Output.** Push to `warnings[]`: `"Severity inflation: <pct>% of findings tagged 'high' (threshold: 40%)"`.
**Why.** A report where >40% of findings are "high" reads as uncalibrated and gets ignored. Real Rails projects have a small number of true-high findings and a long tail of medium/low. If this fires, the most likely cause is that the synthesis defaulted to "high" because picking between high and medium felt arbitrary. Re-anchor to `rubric.md` on those borderlines.

### C2 — Blocker over-use

**Trigger.** Compute `blocker_pct = (count where severity == "blocker") / total_findings`.
**Threshold.** Warn if `blocker_pct > 0.25`.
**Output.** Push to `warnings[]`: `"Blocker over-use: <pct>% of findings tagged 'blocker' (threshold: 25%)"`.
**Why.** Same shape as C1 but with a tighter threshold — "blocker" is the rarest severity. If >25% of findings are blockers, either the project is *truly* on fire (rare, and the report should explicitly say so in the verdict) or the synthesis is inflating. Per `rubric.md`, blockers require a *current-day* exploit/loss/breakage path — "one mistake away" is **high**, not blocker.

### C3 — Unverified blocker

**Trigger.** For each finding where `severity == "blocker"`:

```bash
sed -n '<line>p' '<file>'  # or apply the line_range
```

The output must contain a meaningful subset of `evidence.normalized` (or the verbatim `evidence.snippet` after normalization).

**Threshold.** Any blocker that fails the sed-confirm check.
**Output.**
- `findings[<id>].self_check.status` = `"unverified"`
- Add `<id>` to `self_check.unverified_blockers[]`
- Push to `warnings[]`: `"Unverified blocker: <id> at <file>:<line> — could not confirm cited line. Treat with skepticism."`

**Why.** Agents hallucinate line numbers, especially when summarizing across multiple files. Blockers carry the highest fix-priority weight in the report, so an unverified blocker that turns out to be a hallucination wastes engineering time and erodes trust in the skill. The sed-confirm check is cheap and catches ~all hallucinations.

**Limitation.** A finding whose location is "directory-level" (e.g. `app/services/`) without a specific line number is *not* sed-checked — those findings should not be blockers in the first place; if they are, treat that as a separate signal that synthesis was lazy.

### C4 — Phase dependency mismatch

**Trigger.** For each `fix_sequence[].finding_ids[]` entry, look up the finding by id and confirm `findings[id].phase == fix_sequence_entry.phase`.
**Threshold.** Any mismatch.
**Output.** Push to `warnings[]`: `"Phase dependency mismatch: finding <id> is tagged phase <a> but referenced in fix_sequence phase <b>."`
**Why.** Synthesis may produce a finding tagged `phase: 3` but reference it in `fix_sequence[1].finding_ids` — meaning the human-readable rationale and the per-finding tag disagree. Either is wrong; the writer needs to pick one before the report ships.

### C5 — Scorecard ↔ finding-count mismatch

**Trigger.** For each dimension, compute `weighted_finding_count` (blocker=4, high=3, medium=2, low=1) for findings where the dimension appears in `primary_dimension` OR `secondary_dimensions[]`. Map the count to an expected score band:

| Weighted count | Expected score |
|---|---|
| 0           | 9–10 |
| 1–2         | 7–8 |
| 3–5         | 5–6 |
| 6–10        | 3–4 |
| 11+         | 0–2 |

**Threshold.** Warn if `|expected_score - actual_score| > 2`.
**Output.** Push to `warnings[]`: `"Scorecard mismatch on <dimension>: actual=<a>, expected band=<b>–<c> based on <weighted> weighted finding(s)."`
**Why.** Scorecards anchor the report's narrative arc. If a dimension scores 8/10 but has 6 findings tagged to it, readers get confused. Either the score should drop or the findings need re-tagging. The check is intentionally lenient (>2 points) to allow editorial judgment.

### C6 — Stale coverage data (added v0.3)

**Trigger.** Read mtime of `coverage/.last_run.json` and `coverage/.resultset.json` at audit time.
**Threshold.** Warn if either file's mtime is **>30 days old**.
**Output.**
- Push to `warnings[]`: `"Stale coverage data: coverage/.resultset.json last updated YYYY-MM-DD (<N> days ago). Coverage-derived findings marked unverified."`
- For every finding tagged `test-coverage` whose `tool_origin == "simplecov"`: set `findings[<id>].self_check.status = "unverified"`.

**Why.** Both v0.2 dogfood projects (influapp + coba) had `coverage/.resultset.json` dated March 2025 — read at audit time on 2026-05-05 as if current, producing misleading 0% on payments services that almost certainly have working specs. A 30-day TTL catches the stale case without false-positive on recent runs.

**Skipped.** If neither file exists, the check is silently skipped (no false positive on projects that haven't run SimpleCov locally — those just won't have coverage findings).

**Calibration.** 30-day threshold is a v0.3 first pass. Revisit after 5+ projects' worth of run-cadence data.

### C7 — Real-distribution override (added v0.3)

**Trigger.** When C1 (severity inflation) or C2 (blocker over-use) would fire, evaluate the override:

```
override_applies =
  self_check.unverified_blockers.empty? AND
  summary.risk_score <= 4 AND
  scorecards.count { |s| s.score <= 4 } >= 6
```

**Threshold.** All three conditions must hold.
**Output.**
- If the override applies, **suppress** the C1/C2 warnings. Emit instead: `"Severity distribution reflects real defect density (calibration override C7 applied: <N> sed-verified blockers, risk_score <S>, <M> dimensions ≤4)."`
- The override does **not** affect C3 (unverified blocker) — that always fires when there are unverified blockers.

**Why.** Both v0.2 dogfood projects fired C1 (47% / 48% high). On closer read this reflected real defect density — influapp had 8/8 sed-verified blockers across 8 dimensions ≤4 risk; coba had 4/4 across similar breadth. The C1 warning in those cases is *misleading*: it suggests synthesis was lazy when the project is genuinely on fire. C7 distinguishes "project-on-fire" (override) from "synthesis-inflated" (no override → warning fires as designed).

**Anti-pattern guarded against.** A project might game the override by inflating low scorecard scores. The 6-of-18 dimensions ≤4 threshold is intentionally hard to game without the actual evidence to back it up.

**Calibration.** Revisit after 5+ audits. Likely candidates for adjustment: the `risk_score <= 4` cutoff (could move to 5), the dimensions-≤4 count (could move to 5 or 7).

---

## Output schema

The relevant JSON shape (already in `schema/report.schema.json` as of PR#1):

```json
"self_check": {
  "calibration": {
    "blocker_pct": 0.32,
    "high_pct":    0.48,
    "warnings": [
      "Severity inflation: 48% of findings tagged 'high' (threshold: 40%)",
      "Blocker over-use: 32% of findings tagged 'blocker' (threshold: 25%)"
    ]
  },
  "verified_blockers":   ["f-..."],
  "unverified_blockers": ["f-..."]
}
```

And per-finding:

```json
"self_check": {
  "status": "unverified",   // verified | unverified | demoted | promoted
  "notes":  "sed-confirm at <file>:<line> did not find evidence.normalized substring"
}
```

---

## Render rules

`output-template.md` already conditionally renders the `## Self-check` section above the punch list when `self_check.calibration.warnings` is non-empty. v0.2 does not need template changes.

If the user invokes prompt **P7** (`prompts.md`) interactively, the skill can:
- `show` — print the warnings + affected finding ids
- `demote` — automatic severity drop on findings tagged for inflation (v0.3 only — no-op in v0.2)
- `accept` — continue with warnings noted in report

In v0.2 the default flow is non-interactive: warnings render, no demotion happens, report ships.

---

## What this dimension deliberately does *not* do

- **Does not edit findings without consent.** v0.4 may demote a severity but only when the user explicitly picks `demote` at prompt P7.
- **Does not silently block.** When self-check blocks, the user gets prompt P7 with three explicit choices.
- **Does not check fact-correctness of findings.** That's the job of dimension files + agent fan-out + verification in Step 4. Self-check only audits the *shape* of the report.
- **Does not reconcile scorecards across audit runs.** Trend is a separate dimension.

---

## Future hardening (v0.4+)

After v0.3 lands C6 (stale coverage) and C7 (real-distribution override) and we've run audits against 5+ projects:

1. Promote C1/C2 to **block** when above threshold (after C7 override), with an interactive demote-or-override prompt.
2. Add an "auto-demote" flow: a finding flagged for inflation gets its severity dropped one tier, and `self_check.status: "demoted"` records the change with reasoning.
3. Add C8: severity ↔ time-estimate consistency (a blocker tagged `time_estimate: 5m` is suspicious — either it's overrated or the fix really is trivial).
4. Add C9: cross-dimension consistency — if a finding tagged `secondary_dimensions: [money-and-payments]` doesn't appear in any money-and-payments scorecard reasoning, flag it.

## Note on v0.3 numbering

C6 in v0.3 is **stale coverage data**. The earlier v0.2 placeholder mentioned a "C6 severity ↔ time-estimate consistency" — that's been renumbered to C8 in the v0.4+ plan above. C7 in v0.3 is **real-distribution override**.
