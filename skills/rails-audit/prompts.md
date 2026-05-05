# Interactive prompts

Every user-facing question the skill asks is documented here. Each entry specifies the question text, accepted answers (with aliases), the default, and the fallback for non-canonical input. The skill MUST follow these specs verbatim — do not improvise prompts inline in `SKILL.md`.

## Conventions

- **Case-insensitive** for answers unless explicitly noted.
- **Trim whitespace** before matching.
- **Aliases** map to a canonical answer.
- **Default** is what's chosen if the user hits enter or replies with something equivalent ("yes", "ok").
- **Fallback** is what to do when the input doesn't match any accepted answer — almost always: re-ask once, then proceed with the default if still ambiguous.
- Use the `AskUserQuestion` tool when the input is structured (single choice from a fixed set). Use plain text prompts when the input is free-form (e.g., "Reason for ignore?").

---

## P1 — Mode selection (Step 0, asked when user runs `/rails-audit` with no mode argument)

> "Which mode? **Quick** (~3 min, static-only), **Standard** (~15 min, full audit, default), or **Deep** (~30 min+, includes spec subset)? \[Q/S/D]"

| Answer | Aliases | Maps to |
|---|---|---|
| `S` | `standard`, `default`, `<empty>`, `s` | `standard` |
| `Q` | `quick`, `q`, `fast` | `quick` |
| `D` | `deep`, `d`, `full` | `deep` |

**Default:** `standard`.
**Fallback:** if input matches no alias, re-ask once with the answers spelled out: "Type Q, S, or D." Second non-match → assume `standard` and continue.

---

## P2 — Tool provisioning (Step 0, per missing required tool)

> "**`<tool>`** is required for the audit but isn't installed. Add it where? **Gemfile** (recommended, version-locked), **global** (`gem install`, fast but unpinned), or **skip** (audit continues with reduced confidence)? \[gemfile/global/skip]"

| Answer | Aliases | Behavior |
|---|---|---|
| `gemfile` | `g`, `gem`, `add`, `<empty>` | Append to `:development` group, run `bundle install`, verify `<tool> --version`. Leave the Gemfile change unstaged for the user to commit. |
| `global` | `gi`, `install`, `system` | Run `gem install <tool>`. Note in report appendix that the version is unpinned. |
| `skip` | `s`, `no`, `n`, `later` | Continue without this tool. Note in report appendix; affected dimensions get a confidence-reduced disclaimer. |

**Default:** `gemfile`.
**Fallback:** re-ask with the three options spelled out. Second non-match → assume `gemfile`.

**Hard floor:** if the user picks `skip` for *all of* `rubocop`, `brakeman`, AND `simplecov`, abort the audit. Print: "Without at least one of rubocop/brakeman/simplecov, synthesis has nothing reliable to synthesize. Install one globally with `gem install <tool>` and re-run."

---

## P3 — Recommended tool batch (Step 0, asked once if any Tier-2 tools are missing)

> "These recommended tools are missing: `<list>`. Add them all to the Gemfile now? \[Y/n]"

| Answer | Aliases | Behavior |
|---|---|---|
| `Y` | `y`, `yes`, `<empty>` | Batch-add to `:development` group, run `bundle install`. |
| `n` | `no`, `skip`, `later` | Continue. Each missing tool noted in report appendix. |

**Default:** `Y`.
**Fallback:** assume `Y` after one re-ask.

---

## P4 — Project profile init (Step 0, asked on first run if no `.claude/rails-audit.yml`)

> "I detected: deploy_target=`<inferred>`, job_adapter=`<inferred>`, auth_strategy=`<inferred>`. Confirm and write to `.claude/rails-audit.yml`? \[Y/n/edit]"

| Answer | Aliases | Behavior |
|---|---|---|
| `Y` | `y`, `yes`, `confirm`, `<empty>` | Write `.claude/rails-audit.yml` with the inferred values. |
| `n` | `no`, `skip` | Continue without persisting; auto-detect again next run. |
| `edit` | `e`, `change` | Show the inferred values and ask which field to change (free-form follow-up). |

**Default:** `Y`.
**Fallback:** assume `Y` after one re-ask. If user picks `edit`, follow up: "Which field — deploy_target, job_adapter, or auth_strategy?" Then update only that field.

---

## P5 — Pre-fan-out budget confirmation (Step 0, asked if `--budget` is unset and estimated cost > 30K input tokens)

> "Estimated cost: ~`<input>` input / `<output>` output tokens (`<rough_dollar>`). Continue? \[Y/n/budget=N]"

| Answer | Aliases | Behavior |
|---|---|---|
| `Y` | `y`, `yes`, `<empty>` | Proceed with no budget cap. |
| `n` | `no`, `cancel`, `abort` | Abort the audit. |
| `budget=N` | `b=N`, `cap N`, `limit N` | Set `--budget=N` and proceed. Aborts agent calls that would exceed. |

**Default:** `Y`.
**Fallback:** if input is a number alone (e.g., "50000"), interpret as `budget=50000`. Otherwise assume `Y` after one re-ask.

---

## P6 — Ignore reason (when user asks the skill to suppress a finding interactively)

> "Suppress finding `<id>` (`<title>`)? Reason (free-form, required for `.audit-ignore.yml`):"

Free-form text input. The skill writes:

```yaml
- id: <id>
  reason: "<the user's text>"
  acknowledged_by: <git config user.email>
  expires_at: <today + 90 days>     # ask follow-up: "Expire after N days [90]?"
```

**Validation:** reason must be non-empty and non-whitespace. If empty, re-ask: "A reason is required so future audits know why this is acknowledged."

---

## P7 — Self-check **block** (v0.4+, Step 5.5, when C1/C2 fire AND C7 doesn't override)

> "Self-check is blocking: \<N\> calibration warnings (e.g., \<first warning\>). The skill won't ship the report until you decide. **Block** (default — investigate findings, re-run later), **demote** (auto-demote inflated severities one tier and ship), or **accept** (record an override reason and ship)? \[block/demote/accept]"

| Answer | Aliases | Behavior |
|---|---|---|
| `block` | `b`, `stop`, `<empty>`, `n` | Skill aborts at Step 5.5. Partial JSON written to `tmp/rails-audit/report-YYYY-MM-DD-blocked.json` for inspection. Markdown not rendered. User fixes findings (or runs `--continue` after fixes) and re-runs. |
| `demote` | `d`, `fix`, `auto` | Skill auto-demotes one tier (high → medium for C1; blocker → high for C2). Each demoted finding gets `self_check.status: "demoted"` with a `notes` line citing the check. Skill proceeds to render. |
| `accept` | `a`, `keep`, `ignore` | Skill prompts for a free-form reason (required, non-empty), records it in `audit.calibration_overrides[]`, and proceeds. |

**Default:** `block`.
**Fallback:** re-ask once with the three options spelled out. If still ambiguous, default to `block`.

**v0.2/v0.3 behavior** (warn-only) — the prompt was `show/demote/accept` with default `show`. If a user is on v0.2/v0.3, the skill never blocks regardless of the answer. v0.4 changes the default to `block` and removes the `show` option (P7 is now a decision prompt, not a discovery prompt).

**Auto-demote algorithm** (when user picks `demote`):
- For C1 (severity inflation): identify findings tagged `high` whose evidence/explanation is the weakest (heuristic: shortest `evidence.snippet`). Demote them one tier (`high` → `medium`) until `high_pct ≤ 0.40`. Sort to remove findings without time_estimate first.
- For C2 (blocker over-use): identify findings tagged `blocker` with `time_estimate: "5m"` or `"1h"` (small fixes are rarely true blockers). Demote one tier (`blocker` → `high`) until `blocker_pct ≤ 0.25`.
- A finding can be demoted at most once per audit. The `self_check.notes` records the original severity for trend tracking.

**`accept` reason format** (when user picks `accept`):

```yaml
# audit.calibration_overrides[]
- check_id: "C1"               # or C2
  override_reason: "..."        # the user's text
  override_at: "2026-05-05T15:00:00Z"
  override_by: <git config user.email>
```

---

## Anti-patterns

Avoid these in any prompt:

- **Open-ended yes/no without a default.** Always offer a default; users should be able to enter through the prompt.
- **Stacking questions.** One question at a time. If you need three answers, ask three times.
- **Hidden side effects.** A prompt's answer should map to one well-defined action. Never bundle "and also commit the Gemfile change" into a "yes."
- **Unbounded retries.** At most one re-ask. After that, fall back to the default and continue.
- **Emoji or filler.** "✨ Great choice!" — cut. State the action, take it.

---

## Adding a new prompt

When introducing a new interaction in the skill:

1. Add an entry here following the structure above.
2. Reference it from `SKILL.md` by its `Pn` ID — never inline the prompt text.
3. Include answer aliases that cover plausible user replies (yes/y/yeah, no/n/nope).
4. Specify the default — always.
5. Specify the fallback — always.
