# Releasing rails-audit

Short runbook for cutting a release. The goal is keeping `.claude-plugin/plugin.json`, `CHANGELOG.md`, and the git tag in lockstep — the version-drift gotcha documented in `docs/lessons-learned.md` §6 was the smoking gun for needing this.

## Versioning

[Semantic Versioning](https://semver.org/spec/v2.0.0.html), 0.x while pre-1.0:

- **Patch** (`0.5.1` → `0.5.2`) — fixes, doc updates, additive parser/script changes, schema-additive-only changes (new optional fields).
- **Minor** (`0.5.x` → `0.6.0`) — new features, schema additivity is preserved (no existing field changes shape or becomes required), workflow changes that affect what the audit *does*.
- **Major** (`0.x` → `1.0`) — schema breaking change, or "calibration thresholds are stable across N≥4 real projects" milestone.

Schema additivity is the load-bearing rule. If a change *removes* a field, *renames* a field, or *makes an optional field required*, that's a major bump. New optional fields, new enum values added to existing enums, new appendix sections — all minor or patch.

## Pre-release checklist

Run through this before tagging:

1. **`bin/test-parsers` is green.** CI catches this, but verify locally on the release branch.
2. **All open PRs targeting this milestone are merged.** Check `gh pr list --milestone "v<X.Y.Z>"`.
3. **`docs/lessons-learned.md` reflects what changed.** Patch releases that fix a real bug get a §entry.
4. **Schema additivity holds.** Validate at least one prior-version report against the new schema:
   ```sh
   bin/validate-report skills/rails-audit/examples/sample-report.json
   ```
5. **Real-project dogfood (minor releases only).** If the release changes calibration thresholds, the new thresholds get applied to influapp + coba + pouch reports and the result documented.

## Bumping the version

Three files must move together:

| File | What to change |
|---|---|
| `.claude-plugin/plugin.json` | `"version": "X.Y.Z"` |
| `CHANGELOG.md` | Move `## [Unreleased]` content into a new `## [X.Y.Z] — YYYY-MM-DD` heading; add empty `## [Unreleased]` above it; add the `[X.Y.Z]: …` link reference at the bottom |
| `docs/lessons-learned.md` | Bump the title's version range if the release adds substantial structural lessons |

A single commit captures all three:

```sh
git checkout -b release/v0.5.2
# edit the three files
git add .claude-plugin/plugin.json CHANGELOG.md docs/lessons-learned.md
git commit -m "Release v0.5.2"
git push -u origin release/v0.5.2
gh pr create --title "Release v0.5.2" --body "See CHANGELOG.md."
```

## Tagging + GitHub release

After the release PR merges:

```sh
git checkout main
git pull
git tag -a v0.5.2 -m "v0.5.2 — bin/validate-report pre-write gate"
git push origin v0.5.2
gh release create v0.5.2 --notes-from-tag
```

The tag is what the marketplace points at. The marketplace listing in `kurenn/marketplace` uses the source URL `https://github.com/kurenn/rails-audit.git`; clients pull `main` (or a pinned ref). Tagging makes the release findable but does not by itself update the marketplace — see "Marketplace publishing" below.

## Marketplace publishing

The marketplace at `kurenn/marketplace` is the primary install path:

```bash
claude plugin marketplace add kurenn/marketplace   # one-time per user
claude plugin install rails-audit@kurenn           # one-time install
```

The marketplace's `.claude-plugin/marketplace.json` lists `rails-audit` with the GitHub source URL. Clients fetch the repo head; no marketplace-side change is needed for ordinary releases. **Update marketplace.json only when the listing's metadata changes** — name, description, source URL, category. Version bumps in `plugin.json` are picked up automatically.

## Triage after release

- Skim `gh issue list --label "bug"` for anything the release might have surfaced.
- If the release added a `dimensions/*.md` file or new bin script, add a one-line note to `README.md`'s "What it covers" or "Standalone tool inventory" sections.
- If a calibration threshold moved, the README's `## Calibration status` section needs the new evidence.

## What to avoid

- **Don't bump `plugin.json` without a `CHANGELOG.md` entry.** They drift, marketplace clients see stale versions, and the v0.3.0-vs-CHANGELOG-0.5.1 mismatch (the gotcha that spawned this doc) recurs.
- **Don't tag without merging the release PR first.** Tags should point at commits on `main`, not at branch heads that may or may not land.
- **Don't release on a Friday.** `bin/scan-secrets` has a strict mode; if it fires for the first time on a Saturday morning re-run, you'll be the one investigating.
