# Trend tracking

Per-finding diff against the most recent prior audit report. Replaces v0.1's "score deltas only" trend section with actionable lists of what was *fixed*, what's *new*, and what's *persisted* — so the team can answer "what changed since last audit?" in one glance.

Runs as **Step 6.5** in `SKILL.md` — between self-check (5.5) and markdown render (6). Trend computation operates on the JSON; populates `trend{}`; renders into the `## Trend` section of the markdown report.

---

## Source of truth

Trend reads the most-recent `report-*.json` in `tmp/rails-audit/` whose date is older than the current run. Selection algorithm:

1. List `tmp/rails-audit/report-*.json` (sorted by filename — date is in the name).
2. Drop any whose filename date is ≥ today (those are *this* run's prior partial writes).
3. Take the most recent remaining. That's the "prior."
4. If no prior exists, set `trend = null` and skip rendering. Don't render `## Trend (no prior report)` — empty state is just absence.

Prior report must be `schema_version: 2` or higher. v0.1 markdown reports are not compatible (no fingerprints).

---

## Fingerprint diff

The fingerprint algorithm (locked in PR#1) is stable across line moves and whitespace edits but unstable across file renames or material code changes. That makes the diff buckets clean:

```
prior_ids   = Set(prior.findings.map(&:id))
current_ids = Set(current.findings.map(&:id))

fixed_ids     = prior_ids - current_ids
new_ids       = current_ids - prior_ids
persisted_ids = prior_ids ∩ current_ids
```

Stored in JSON:

```json
"trend": {
  "prior_report_path": "tmp/rails-audit/report-2026-04-21.json",
  "prior_report_date": "2026-04-21",
  "fixed_ids":     ["f-..."],
  "new_ids":       ["f-..."],
  "persisted_ids": ["f-..."]
}
```

---

## first_seen propagation

For each persisted finding, copy `first_seen` from the prior report (if present); else set to the prior report's date. New findings get `first_seen = today`. Fixed findings — by definition not in current — don't need first_seen; the dropped finding's history is preserved by its presence in the prior report.

The `first_seen` field is rendered next to persisted findings in the markdown ("first seen 2026-04-21, ~14 days ago"). Persisted findings ≥ 30 days old get a `[stale]` badge — they've been on the punch list for too long.

---

## Ignored findings in trend

Per PR#7's design, findings with `ignored: true` are excluded from the punch list but **included** in trend. So:

- An ignored finding that was fixed → appears in `fixed_ids` (and Appendix D no longer lists it).
- An ignored finding still present → appears in `persisted_ids`.
- A new ignore added since last run → just changes `ignored_findings[]` membership; no trend movement.

This gives the team visibility into "we acknowledged this 6 months ago and it's still here" without re-spamming the punch list.

---

## Scope-aware trend

Per PR#8, scoped audits (`--only=money`) restrict findings to in-scope dimensions. Trend computation also restricts:

- The prior report's findings are filtered to in-scope dimensions before diffing.
- This means a `--only=money` run produces a money-only trend.
- A finding tagged primary `code-smells` with secondary `money-and-payments` is included in both a `code-smells` scope trend AND a `money-and-payments` scope trend.

---

## Per-dimension trend table

The markdown render produces a per-dimension trend table:

```markdown
| Dimension | Fixed | New | Persisted | Score Δ |
|---|---|---|---|---|
| Money & Payments | 2 | 1 | 4 | +1 |
| Security & AuthN | 0 | 2 | 5 | -1 |
| Deploy & CI | 5 | 0 | 1 | +3 |
| ... | | | | |
```

For each row:
- Counts come from the trend bucket sets, filtered by dimension membership (primary OR secondary).
- Score Δ is `current.scorecard.score - prior.scorecard.score` for that dimension.

Below the table, render the *titles* of the most-impactful changes (top 5 fixed, top 5 new) with their fingerprint-derived IDs and locations. "Top" = blocker > high > medium > low.

---

## Stale persisted

A finding persisted across ≥ 30 days warrants attention. The render includes a **Stale persisted** sub-table:

```markdown
> **Stale persisted** (3): findings present for ≥30 days without being fixed or acknowledged.

| ID | Title | Severity | First seen | Days |
|---|---|---|---|---|
| `f-...` | Stripe payouts missing idempotency keys | high | 2026-03-15 | 51 |
| ... | | | | |
```

Stale persisted is signal for one of:
- The team agreed to fix it but hasn't (track in issue tracker).
- The team should acknowledge with `.audit-ignore.yml` (suppress with reason).
- The fix is harder than expected (escalate or re-prioritize).

---

## When trend is rendered

In the markdown report, `## Trend` appears between `## Recommended fix sequence` and `## Appendix A — Tooling`. Renders only when `trend` is non-null. Empty trend (first run) is omitted entirely — no "no prior report" placeholder.

---

## --track-renames mode (added v0.4)

By default, the locked-in fingerprint algorithm is unstable across file renames — a moved finding shows as `fixed + new`. That's the right default: any rename is real code change worth highlighting.

For projects where renames are common (active refactoring, monorepo splits), opt in with **`--track-renames`** in trend computation:

1. For each `fixed_id` (in prior, not in current), look at its prior `primary_dimension` and finding-type (derived from title prefix).
2. For each `new_id` (in current, not in prior), look at the same.
3. Pair them if all of:
   - Same `primary_dimension`.
   - Same finding-type (e.g. "Command Injection", "update_column on …", "Stripe webhook missing …").
   - `evidence.normalized` Levenshtein distance ≤ 20% of the prior length OR ≤ 30 characters absolute, whichever is larger.
4. Each pair becomes a `trend.moved_ids[]` entry and is **removed** from both `fixed_ids` and `new_ids`.
5. The current finding's `first_seen` propagates from the prior — age is preserved.

### Trade-offs

Fuzzy matching can produce false positives. Two unrelated `update_column` findings in the same dimension could match. False positives are visible in the trend table — the user can spot a wrong pair by inspecting `prior_location` vs `current_location`.

### Configuration

`.claude/rails-audit.yml`:

```yaml
track_renames: false       # default — explicit opt-in keeps trends precise
rename_distance_pct: 0.20  # tunable — higher = looser matches
```

CLI override: `--track-renames` forces it on for one run regardless of config.

### Schema (additive)

`trend.moved_ids[]`:

```json
"moved_ids": [
  {
    "prior_id":         "f-9c8a3b...",
    "current_id":       "f-2d4e6f...",
    "prior_location":   { "file": "app/services/payments/payout.rb", "line": 35 },
    "current_location": { "file": "app/services/payouts/processor.rb", "line": 41 },
    "match_score":      0.85
  }
]
```

## What this dimension deliberately does *not* do

- **Does not track renames by default.** Opt-in only.
- **Does not fall back gracefully on schema version mismatch.** If the prior report is `schema_version: 1` (or anything other than the current schema), trend is set to null with `audit.ignore_warnings[]` noting the version skew.
- **Does not measure score velocity.** The trend table shows deltas, not rates of change. Multi-run velocity is a v0.5+ idea if needed.
- **Does not auto-archive prior reports.** Old reports stay in `tmp/rails-audit/` until the user cleans them up. v0.5+ may add a retention policy (keep most recent N).

---

## Calibration

The 30-day stale threshold is a guess. After 5+ real-project audits with multiple runs, revisit:

- Is 30 days too lax? Too strict?
- Should stale-persisted percentage feed back into the self-check (PR#5)? E.g., warn if >50% of findings are stale-persisted — the audit isn't actionable.
- Should there be a "rapidly fixed" badge for findings that disappear within 7 days?
