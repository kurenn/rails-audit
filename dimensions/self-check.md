# Self-check (audit-the-audit)

The skill polices its own report before delivering it. Self-check produces *meta-findings* about the audit itself — not the codebase. Outcomes land in JSON `self_check{}` and render in the markdown report as a `## Self-check` section above the punch list.

In **v0.2**, self-check is **warn-only**: it never blocks. It surfaces calibration smells and lets the user decide. Hardening to block (with auto-demotion) is planned for v0.3 once the thresholds have been calibrated against ~5 real audits.

---

## Run order

Self-check runs as **Step 5.5** in `SKILL.md` — *after* synthesis (Step 4) produces JSON, *before* the markdown render (Step 5). Self-check may mutate `self_check{}` and individual `findings[].self_check{}` fields; it does not mutate severities or remove findings (v0.2 constraint).

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

- **Does not edit findings.** v0.2 self-check is read-only on the synthesis output.
- **Does not block.** Even a 100%-blocker report ships.
- **Does not check fact-correctness of findings.** That's the job of dimension files + agent fan-out + verification in Step 4. Self-check only audits the *shape* of the report.
- **Does not reconcile scorecards across audit runs.** Trend (PR#10) is a separate dimension.

---

## Future hardening (v0.3)

When the v0.2 thresholds have been calibrated against ~5 real audits:

1. Promote C1/C2 to **block** when above threshold, with an interactive demote-or-override prompt.
2. Add an "auto-demote" flow: a finding flagged for inflation gets its severity dropped one tier, and `self_check.status: "demoted"` records the change with reasoning.
3. Add C6: severity ↔ time-estimate consistency (a blocker tagged `time_estimate: 5m` is suspicious — either it's overrated or the fix really is trivial).
4. Add C7: cross-dimension consistency — if a finding tagged `secondary_dimensions: [money-and-payments]` doesn't appear in any money-and-payments scorecard reasoning, flag it.
