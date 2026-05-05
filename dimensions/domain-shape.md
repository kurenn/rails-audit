# Dimension: Domain shape

Orientation only — captures *what the app does* in 1–2 sentences and inventories the layers. Drives nothing else, but every other dimension's findings need this context to be intelligible.

## What "good" looks like

- Domain summary fits in 2 sentences.
- Layering is consistent: models for state + invariants, services for orchestration, forms for input validation, jobs for async, serializers for output, controllers for HTTP only.
- No surprise top-level layer (e.g. `app/orchestrators/` mixed with `app/services/` doing the same thing).
- Routes file is readable in a single screen-scroll (or has an obvious organizing principle).

## Checks

### Domain summary

Read top-level models and routes; write a 1–2 sentence description. Cite the 5–10 most central models. This goes in the report's executive summary section as context.

```bash
ls app/models/*.rb | head -20
wc -l config/routes.rb
head -50 config/routes.rb
```

### Routes inventory

```bash
bundle exec rails routes -E > tmp/rails-audit/routes.txt 2>/dev/null || \
  bin/rails routes > tmp/rails-audit/routes.txt 2>/dev/null
wc -l tmp/rails-audit/routes.txt
```

Categorize routes:
- HTML/web — count
- JSON API — count
- Admin (ActiveAdmin or similar) — count
- Webhooks/callbacks — count
- Health/internal — count

Surface anomalies:
- Routes mounted at root that should be namespaced (`/api/v1/...`)
- API namespace without versioning
- Mass-mutation routes on the unauthenticated tier (e.g. `delete-past-campaigns` exposed without user auth)

### Layer inventory

```bash
ls app/
# What layers beyond MVC are present? Common: services, forms, jobs, serializers, mailers, decorators, queries, policies, validators
```

For each layer, count files + total LOC:

```bash
for layer in models controllers services forms jobs serializers mailers decorators queries policies validators interactors; do
  if [ -d "app/$layer" ]; then
    count=$(find app/$layer -name "*.rb" | wc -l)
    loc=$(find app/$layer -name "*.rb" -exec wc -l {} + 2>/dev/null | tail -1 | awk '{print $1}')
    echo "$layer: $count files, $loc LOC"
  fi
done
```

### Layer consistency

If multiple layers seem to do the same thing (`app/services/` and `app/interactors/`, or `app/forms/` and `app/validators/`), note the overlap. Pick the dominant pattern by file count; flag the minority pattern as a refactor target.

### Concerns inventory

```bash
ls app/models/concerns/ app/controllers/concerns/ 2>/dev/null
```

Each concern: 1-line summary of what it provides. Concerns named after a class (`UserAuthentication`) are usually fine; concerns named after a verb (`Searchable`, `Cacheable`) need state to justify being concerns vs POROs.

### Dependency graph (informal)

If there are >50 models, sketch the top-level domain clusters (e.g. "Users / Brands / Influencers" + "Campaigns / Collaboration Requests / Evidence" + "Payments / Payouts / Commissions"). Each cluster should map to a directory or naming pattern.

## Severity calibration

This dimension rarely produces findings on its own. Use it for context. Flag **Medium** only when:

- Two layers do the same thing (e.g. services and interactors duplicating responsibility) — refactor target.
- API has no version namespace (`/api/...` without `/v1`).
- Mass-mutation routes (delete/refresh/recompute) exposed at the unauthenticated tier.

Otherwise, no findings. Just orient the report.
