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

## P7 — Self-check warning surfaced to user (Step 5.5, when calibration warnings fire)

> "The self-check raised `<N>` calibration warnings (e.g., `<first warning>`). Show details, demote findings, or accept and continue? \[show/demote/accept]"

| Answer | Aliases | Behavior |
|---|---|---|
| `show` | `s`, `details`, `<empty>` | Print the full warning list with affected finding IDs. Then re-ask. |
| `demote` | `d`, `fix`, `auto` | For severity-inflation warnings, the skill auto-demotes findings (high → medium, blocker → high) using its own judgment. Marks `self_check.status: "demoted"` per finding. |
| `accept` | `a`, `keep`, `ignore` | Continue with warnings noted in report but no demotions. |

**Default:** `show`.
**Fallback:** re-ask with the three options spelled out.

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
