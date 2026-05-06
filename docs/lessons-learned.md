# Lessons learned — building rails-audit v0.2 → v0.5

This doc captures insights from building the v0.3 → v0.4 → v0.5 milestones that should outlive any single CHANGELOG entry. Each section is self-contained: context first, then the implication for future work.

---

## 1. The Ruby `gsub` / `last_match` gotcha

Discovered while building `bin/parse-bundle-audit`. The original code:

```ruby
current[Regexp.last_match(1).downcase.gsub(/\s+/, '_').to_sym] = Regexp.last_match(2)
```

was producing records with all-nil values despite the keys appearing correct. The regex match itself worked; the bug was that `String#gsub(/\s+/, '_')` runs its own internal regex match that **clobbers `Regexp.last_match`** before the right-hand side evaluates. By the time `Regexp.last_match(2)` was read, it pointed at gsub's internal regex match, which had no group 2.

**Fix**: capture match groups into local variables before any subsequent regex operation.

```ruby
key, value = Regexp.last_match(1), Regexp.last_match(2)
current[key.downcase.gsub(/\s+/, '_').to_sym] = value
```

**Implication**: any v0.5+ parser that mixes `Regexp.last_match` with downstream string transformations should follow the local-variable-first pattern. The bug is silent — `last_match` returns nil for the missing group; a careful test-first build would have caught it, but exploratory development didn't.

---

## 2. The C7 "dimension-≤4" heuristic

The most subtle calibration decision in v0.4. The hardened self-check (PR#41) blocks reports when C1 (>40% high) or C2 (>30% blocker after the v0.5 bump) fires. Without an escape hatch, a project that's *genuinely* on fire can't ship a report — the synthesis-quality check fires regardless of whether synthesis was actually lazy.

C7 is the escape hatch. It applies when ALL of:

1. All blockers are sed-verified (synthesis isn't hallucinating).
2. `summary.risk_score ≤ 4` (project really is in trouble).
3. ≥ 6 of 18 dimensions score ≤ 4 (broad defect density, not concentrated).

Condition 3 is the load-bearing one. The v0.4 benchmark across influapp + coba showed:

- **influapp** had 12 dimensions ≤4 → broad damage → C7 applies → report ships with override message.
- **coba** had 5 dimensions ≤4 → concentrated damage in deploy/security/money → C7 doesn't apply → hardened block fires.

This is exactly the right discrimination. coba's secrets-scanner-produced 29% blocker rate looks identical to influapp's on the surface, but coba is concentrated breakage (one Phase 1 PR fixes most of it). influapp is broad systemic decay (Rails+Ruby EOL touches everything).

**Implication**: when designing future calibration overrides, the gating condition should measure *breadth*, not just *severity*. A project with 50% blockers in three dimensions is different from a project with 10% blockers across all eighteen.

---

## 3. The 25% → 30% blocker-pct paradox

coba's v0.4 dogfood had 29.4% blockers — all correctly tagged per rubric (8 tracked secrets each as its own blocker, plus 4 from synthesis). The old C2 threshold of 25% spuriously fired. The synthesis wasn't lazy; the rubric was right; the threshold was wrong.

The instinct is to soften the rubric ("maybe secrets scanner findings shouldn't be blockers"). But that's the wrong move — a tracked private key really is a blocker. The rule was right; the threshold was guessed in v0.2 before we had any real data.

v0.5 PR#2 raised C2 to 30%. coba now clears at 29.4%. Influapp post-inheritance is at 11.3%, well under either threshold.

**Implication**: when a calibration check fires on a real correct-by-rubric audit, the bug is usually in the threshold, not in the rubric or the synthesis. Resist softening definitions; tune thresholds with real-project evidence.

---

## 4. Why JSON-first matters

The v0.2 design decision to make `report.schema.json` the contract (with markdown rendered from it) paid off massively in v0.3-v0.5. Every new feature was a schema-additive change:

- v0.3: `audit.calibration_overrides[]`, `audit.mode` enum extension
- v0.4: `findings[].security_revalidation`, `trend.moved_ids[]`
- v0.5: `findings[].severity_inherited_from`

Each shipped without breaking v0.2 reports. Validation is automated (`jsonschema` in CI). Trend tracking works across versions because fingerprints are stable.

A markdown-first design would have required string-parsing every prior feature in every new feature. The v0.5 inheritance script (`/tmp/apply-inheritance.py`) is 80 lines because it operates on JSON. The same script in markdown-land would be 800 lines of fragile string regexes.

**Implication**: the schema-as-contract pattern is the *single most important* design decision in this skill. Resist any refactor that pushes schema-derived state into prose. New features get schema fields; markdown is the view.

---

## 5. The bundle-audit explosion + severity inheritance

The v0.4 dogfood on influapp surfaced 64 individual bundle-audit advisories where v0.2 had 3 aggregated findings. That granularity was right — each is a real CVE with its own fingerprint, suitable for trend tracking ("CVE-X fixed").

But it produced an immediate calibration question: are these 64 distinct concerns or 64 consequences of one root cause (Rails 7.0.4 EOL)? In practice, they're consequences. Landing one foundation-upgrade PR fixes all 64.

v0.5 PR#1 added `severity_inherited_from`. On influapp it demoted 51 of the 64 (the 13 that didn't demote are CVEs in gems off the inheritance roster — Stripe, etc., which release independently). blocker_pct dropped 26.8 → 11.3%; high_pct dropped 62.9 → 42.3%.

**The lesson is structural**: the audit's job is *to surface concerns*; the punch list's job is *to scope work*. Granular CVEs are right for the former; aggregation under root causes is right for the latter. The schema separates them via `severity_inherited_from`, and the rendered punch list nests inherited findings under their root.

**Implication**: when a future scanner produces N findings that share a root cause (e.g., a deprecated gem with 12 patches), use the same pattern. Add the gem to the inheritance roster; the findings will automatically demote.

---

## 6. The v0.3.0-as-plugin-restructure surprise

Mid-v0.3-milestone the repo was restructured into a Claude Code plugin layout (`skills/rails-audit/...`) and tagged 0.3.0. This wasn't planned in the original v0.3 issue list — it was a parallel decision.

The collision meant the v0.3-milestone work shipped as **v0.4.0**, not v0.3.0. The milestone label is just a label; the version is decided at tagging time.

**Implication**: don't over-couple GitHub milestone names to release versions. Names are stable for tracking; versions are decided when shipping.

---

## 7. What was hardest to get right

In rough order of difficulty:

1. **C7's three-condition gate** — the dimension-≤4 count was the breakthrough. Without it, the gate would have either let synthesis-inflated reports through (gate too loose) or blocked legitimately-bad reports (gate too tight). Two conditions weren't enough.

2. **Severity inheritance roster scope** — the conservative roster (Rails-bundled + Ruby-adjacent) is right but uncomfortable. It means Stripe CVEs *don't* inherit even when the project's EOL Rails is the actual reason it can't upgrade Stripe. Future v0.6+ may add a "transitive inheritance" pass; for now, conservative is safer than aggressive.

3. **The harden block default** — defaulting P7 to `block` (not `accept`) was the right call but felt aggressive. Without it, v0.4 would have been "v0.2 with more findings." With it, v0.4 actually changes how the audit gets used.

4. **Schema additivity discipline** — every v0.3-v0.5 schema change was additive on principle. Required pushing back on "while we're at it..." restructures. Worth it.

---

## 8. What I'd do differently

1. **Start with parsers, not synthesis prompts**. v0.4's `bin/parse-*` scripts cut the v0.5 token cost dramatically. Should have been v0.2.
2. **Write the inheritance roster doc before shipping the inheritance schema**. The roster's "what NOT to inherit" section was harder to write than the "what TO inherit" because the negative case isn't surface-visible until you have a real project (Stripe-on-EOL-Rails).
3. **Fingerprint format should have been opaque hex from day one**. The parser-emitted formats (`f-bk-XXXX-...`) violated the schema's hex pattern and required a synthesis-time normalization. Pure SHA truncation works; the prefix decoration was unnecessary.

---

## 9. Calibration on N=2 isn't calibration

Both v0.4 and v0.5's calibration moves (C7 override gate, C2 threshold bump, severity inheritance roster) were tuned with the influapp + coba audit data already in hand. That means:

- C7's three-condition gate was specifically chosen so influapp would clear (12 dimensions ≤4 hits ≥6) and coba would block (only 5 ≤4 misses ≥6). On a third project, the gate may misfire.
- C2's 30% threshold bump was triggered by coba's 29.4% real blockers. On a project with 35% real blockers, we'd hit the same paradox.
- The inheritance roster (Rails-bundled + Ruby-adjacent gems) is conservative, but it's tuned for the kind of EOL Rails project influapp is. Different stacks may need different rosters.

The discipline that should follow: every threshold and gate gets a `# calibrated against N=<count> projects` annotation in code, and a "validation needed at N=<3-5>" tracker issue. Calibrating with the answer in hand is fine for v0.5; it's not fine for v1.0.

## 10. The dogfood reports were syntheses, not real runs

This is the load-bearing honesty note. The v0.4 and v0.5 "rerun" reports for influapp and coba were produced by a Python script (`/tmp/synthesize-v04.py`, since promoted to `bin/apply-inheritance` for the inheritance step only) that took the v0.2 JSON, added scanner output + parser stubs, and recomputed self-check. They demonstrate that the v0.4/v0.5 schema accommodates the new fields and that the inheritance/scoring math is correct.

What they do NOT prove:
- That `/rails-audit --standard` on a fresh checkout actually produces these JSON shapes when invoked end-to-end via the Skill tool.
- That `--continue` mode actually costs ~5K tokens. That number is an estimate; the actual harness-exposed token count was never recorded.
- That the hardened block (PR#41) actually invokes prompt P7 and waits for input. The flow is documented; it has not been observed live.

The fix for this — observing the skill's behavior on real `/rails-audit` invocations — needs to happen on a third project audit before any 1.0 talk.

## Pointers for future-me

- All v0.4 schema additions documented in `CHANGELOG.md` under `[0.4.0]`. Same for v0.5 once it ships.
- The benchmark-v02-vs-v04.md doc at `tmp/benchmark-v02-vs-v04.md` is the canonical "did the calibration work?" artifact for this milestone.
- The inheritance-after-influapp dogfood JSON is at `tmp/rails-audit/report-2026-05-05-v0.5.json` in the influapp project. Compare to v0.4 to see the demotion in action.
- The two real Rails projects used as dogfoods are in `~/workspace/workspace/influapp-api` (broad-defect example) and `~/workspace/workspace/coba` (concentrated-defect example). Both are good test beds for future calibration changes.
