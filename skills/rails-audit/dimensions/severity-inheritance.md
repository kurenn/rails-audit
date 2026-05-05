# Severity inheritance

When N findings are consequences of one root-cause finding, severity inheritance demotes them and links them to the root. The trend tracks the root finding's lifecycle; the inherited findings ride along.

Runs as **Step 4.7** in `SKILL.md` — after revalidations (4.5, 4.6), before self-check (5.5).

---

## Why this exists

The v0.4 dogfood on influapp surfaced 17 bundle-audit blockers + 46 highs. All 63 were CVEs in actionpack / actionview / loofah / rack / etc. — every one inherited from Rails 7.0.4 being EOL. Each is technically real, but landing one foundation-upgrade PR fixes all 63 in a single shot.

Without inheritance, the punch list reads as 63 distinct concerns and the team is overwhelmed. With inheritance, those 63 nest under a single B1 (Ruby/Rails EOL) finding and the work scope is honest.

---

## Detection rules

A finding is inherited when ALL of these hold:

1. **Inheritance-eligible source.** Currently: `tool_origin == "bundle-audit"`. (Future v0.6+ may add brakeman warning chains.)
2. **Foundation tag.** Finding has `foundation` in `primary_dimension` OR `secondary_dimensions`.
3. **Stack-coverage match.** The CVE's gem is on the inheritance roster (see below).
4. **Active root-cause finding.** A finding with `tool_warning_id: "unmaintained_dependency"` exists in the same report covering the relevant version (Ruby or Rails on EOL).

If any of (1)-(4) fails, the finding is NOT inherited — it stands on its own severity.

### Inheritance roster (Rails / Ruby)

These gems' CVEs inherit from a Rails or Ruby EOL root-cause finding when one exists. The list is conservative — only gems whose CVE patches are bundled into the Rails/Ruby release cycle.

```
Rails-bundled gems (CVE inherits from Rails EOL):
  rails, railties, actionmailer, actionmailbox, actionpack, actionview,
  actiontext, actioncable, activerecord, activemodel, activestorage,
  activejob, activesupport, sprockets-rails

Ruby-bundled / Rails-adjacent (CVE inherits from Ruby OR Rails EOL):
  rack, rack-test, nokogiri, loofah, sanitize, mail, json, uri,
  rexml, cgi, openssl

Build-chain (rarely affected, but inherits if covered):
  bundler, thor, rake
```

NOT in the roster (treat as standalone findings even if Rails is EOL): `stripe`, `aws-sdk-*`, `redis`, `pg`, `mysql2`, `sidekiq`, `cloudtasker`, `puma`, `devise`, `pundit`, `image_processing`. These have their own release cadences independent of Rails.

---

## Action

For each finding that meets all four conditions:

```
findings[<id>].severity_inherited_from = <root_id>
findings[<id>].severity = demote_one_tier(findings[<id>].severity)
findings[<id>].self_check.notes ||=
  "Demoted by severity inheritance from <root_id> (foundation upgrade fixes this)."
```

`demote_one_tier`: blocker → high → medium → low → low (low is the floor).

The root-cause finding is **not** modified — it remains at its native severity (typically blocker for an EOL Rails).

---

## When NOT to inherit

Even when a finding looks like a candidate, **skip inheritance** if:

- The root-cause finding has been **ignored** via `.audit-ignore.yml`. Without an active root, the children can't ride along.
- The finding's CVE has a **patch available in the current Rails minor** (i.e., the gem is on the roster but the patched version exists in the current Rails branch). In this case the team can patch without a foundation upgrade — the finding stands on its own.
- The finding's `severity` is already `low`. Demotion floor reached.
- The finding has been **manually annotated** with `inherit: false` (escape hatch for cases where the team wants to track it independently — e.g., a CVE they're patching ahead of the foundation upgrade).

---

## Render

Inherited findings do NOT appear as top-level punch-list entries. Instead, they render as a sub-list under the root-cause finding:

```markdown
#### B1. Ruby and Rails are both EOL — Foundation · Security · Deploy & CI
**Location:** `.ruby-version:1`
**Evidence:** `ruby-3.1.2`
**Why this is a blocker:** ...
**Fix sketch:** Plan upgrade to Ruby 3.3+ and Rails 7.1/7.2 LTS.

> **Inherited findings** (63): the following CVEs are consequences of this root-cause and resolve when the foundation upgrade lands. They appear in the JSON `findings[]` for trend tracking but are not separate punch-list entries.
>
> | CVE | Gem | Severity (post-demotion) |
> |---|---|---|
> | CVE-2024-47889 | actionmailer | medium (was high) |
> | CVE-2023-22792 | actionpack | high (was blocker) |
> | ... | ... | ... |
```

The summary line ("63 inherited findings") preserves the work-scope visibility without flooding the punch list.

---

## Trend interaction

When trend computes between a v0.5+ prior and current report:

- **Root finding fixed** → all inherited findings auto-resolve. Trend lists the root in `fixed_ids`; inherited children are NOT counted in `fixed_ids` (they're consequences, not separate fixes).
- **Root finding persists** → inherited findings persist (their fingerprints are stable across runs).
- **A new inherited finding appears** (a new CVE published since prior) → it's listed in `new_ids` AND linked to the root. The trend's "new" count includes inherited new findings.

This makes the trend table honest: foundation-upgrade work shows up as a single root being fixed, not as "63 fixes."

---

## Calibration

- **Roster completeness**: the v0.5 roster is conservative. After 5+ audits across diverse Rails projects, expand if false negatives accumulate (CVEs that should be inherited but aren't on the list).
- **Brakeman as a future inheritance source**: if Brakeman flags a "Default Routes" warning that's a consequence of an Active Admin version, that's an inheritance candidate too. v0.6+.
- **Demotion tier-count**: currently always one tier. If real audits show that 50+ inherited findings still produce noisy punch lists post-demotion, consider a 2-tier demotion for very-high-count inheritance chains (>20 children).

---

## What this dimension deliberately does NOT do

- **Does not fix anything.** It's purely a presentation/calibration layer over the synthesis output.
- **Does not modify the root finding.** Root stays at native severity.
- **Does not auto-detect non-Rails inheritance chains** (e.g., Sidekiq-bundled gems). Roster is conservative; manually-tagged `inherit: false` is the escape hatch.
- **Does not affect coverage gates.** Critical-path coverage requirements are independent of severity (a payment service still needs ≥80% coverage even if its CVEs are inherited).
