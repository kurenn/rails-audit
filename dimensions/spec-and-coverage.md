# Dimensions: Spec stability + Test coverage

Cluster A. Audit the test suite for stability and the coverage signals that matter — not raw % on a dashboard.

## What "good" looks like

- Specs run in random order, retries off, no flake. Failed runs reproduce locally with the same seed.
- Critical paths (`app/services/payments/`, `app/controllers/webhooks/`, auth) at ≥80% **branch** coverage.
- No `pending`/`skip`/`focus` left in tree.
- External services (Stripe, Twilio, GCS, Cloud Tasks, identity providers) stubbed via WebMock or a shared fake; zero real network calls in CI.
- Factories don't cascade — creating one record doesn't accidentally create five.
- CI captures screenshots, logs, and the seed on failure. Runtime bounded.

## Checks

### Coverage breadth

```bash
ls spec/ 2>/dev/null
find spec -name "*_spec.rb" | wc -l
find spec -type d -mindepth 1 -maxdepth 1
# Per-layer counts
for d in models requests services jobs system forms mailers channels routing controllers; do
  echo "$d: $(find spec/$d -name "*_spec.rb" 2>/dev/null | wc -l)"
done
```

For each layer in `app/`, list files **without** a corresponding spec, sorted by LOC descending. The largest unspec'd files are the highest-leverage gaps.

```bash
# Largest unspec'd services
for f in app/services/**/*.rb; do
  base=$(basename "$f" .rb)
  spec_path=$(echo "$f" | sed 's|app/|spec/|; s|\.rb$|_spec.rb|')
  test -f "$spec_path" || echo "$(wc -l < "$f") $f"
done | sort -rn | head -20
```

### Flakiness signals

Grep patterns to surface:

```bash
# Time without freezing
grep -rn "Time\.\(now\|current\)\|Date\.today\|DateTime\.now" spec/ --include="*.rb" | grep -v "freeze_time\|travel_to\|Timecop"
# Sleep
grep -rn "sleep " spec/ --include="*.rb"
# Random data without seed control
grep -rn "rand\|SecureRandom" spec/ --include="*.rb" | grep -v "Faker"
# Real network
grep -rn "VCR\|WebMock\|stub_request" spec/ --include="*.rb" | wc -l
grep -n "disable_net_connect" spec/spec_helper.rb spec/rails_helper.rb 2>/dev/null
# Pending / focus / skip
grep -rn "pending\|^[[:space:]]*xit\|^[[:space:]]*xdescribe\|^[[:space:]]*xcontext\|:focus\b" spec/ --include="*.rb"
# allow_any_instance_of (brittle)
grep -rn "allow_any_instance_of" spec/ --include="*.rb" | wc -l
```

### Configuration

Read these in full:
- `.rspec`
- `spec/spec_helper.rb`
- `spec/rails_helper.rb`
- Every file in `spec/support/` (one-line summary of what each does)

Look for:
- Random ordering on (`config.order = :random`) — good
- `--retry` plugin (`rspec-retry`) — note presence/absence
- Transactional fixtures or DatabaseCleaner — at least one
- WebMock with `disable_net_connect!` and a sane allowlist
- Cloudtasker / Sidekiq testing mode set (e.g. `Cloudtasker::Testing.fake!`, `Sidekiq::Testing.fake!`)

### CI mechanics

Read the test workflow file (`.github/workflows/*test*.yml`, `*ci*.yml`):
- Parallelization?
- Retries?
- Fail-fast?
- Artifact upload (screenshots, logs)?
- Image target — is the `tests` service using a `testing` build target or the `development` one (former is faster, leaner)?

### Factories

```bash
ls spec/factories/ 2>/dev/null
wc -l spec/factories/*.rb 2>/dev/null
```

Read each factory. Smell signals:
- Traits that auto-create unrelated associations (cascade)
- Sequences across factories that could collide
- File attachments in default attributes (slow)

### Coverage signals — what to read

If `coverage/.last_run.json` and `coverage/.resultset.json` exist (SimpleCov), parse them:

```bash
cat coverage/.last_run.json 2>/dev/null
# Per-file coverage from .resultset.json — extract the data per file
ruby -r json -e 'data = JSON.parse(File.read("coverage/.resultset.json")); data.each { |suite, run| run["coverage"].each { |path, info| lines = info["lines"] || info; covered = lines.compact.count { |x| x > 0 }; total = lines.compact.size; pct = total > 0 ? (covered.to_f / total * 100).round(1) : 0; puts "#{pct}% #{path}" } }' 2>/dev/null | sort -n | head -30
```

Build the **risk-weighted coverage map**:
- Critical paths (from project profile or auto-detected: `app/services/payments/`, `app/controllers/webhooks/`, auth controllers, `app/services/identity_platform/`): require ≥80% line, ≥70% branch.
- High-risk paths (jobs, services touching external state): require ≥60% line.
- Other paths: report-only, no threshold.

Branch coverage requires SimpleCov configured with `enable_coverage :branch` — note absence as a finding.

### Spec-to-code ratio

```bash
spec_loc=$(find spec -name "*_spec.rb" -exec wc -l {} + | tail -1 | awk '{print $1}')
app_loc=$(find app -name "*.rb" -exec wc -l {} + | tail -1 | awk '{print $1}')
echo "spec/app ratio: $(echo "scale=2; $spec_loc / $app_loc" | bc)"
```

A ratio < 0.5 in a Rails app is a yellow flag (thin tests). Per-layer ratios are more telling than the global one.

## Severity calibration for this dimension

| Pattern | Default tier |
|---|---|
| Critical-path file (payments/auth/webhooks) with 0% coverage | Blocker |
| Critical-path file with <50% line coverage | High |
| `Time.current` / `Time.now` in spec without freeze | High |
| No retry on random-order CI with known flake history | High |
| 8+ controllers without request specs | High |
| Allowlisted real network calls in CI | High |
| `allow_any_instance_of` >10 occurrences | Medium |
| Factory cascade smell | Medium |
| `pending`/`skip` left in tree | Medium |
| No branch coverage configured | Medium |
| Spec/app LOC ratio <0.5 globally | Medium |
| `:focus`/`fit` left in tree | High (unmerged broken commit smell) |

## Cross-cuts

When tagging findings in this dimension, also consider these as `secondary_dimensions[]`:

- **`developer-experience`** — slow CI / unstable specs are also DX problems (CI runtime >15min, flaky retries).
- **`risk-hotspots`** — a large untested file is *both* a hotspot and a coverage gap; tag both.
- **`money-and-payments`** — coverage gaps inside `app/services/payments/*` are money-criticality; tag both.
- **`security-and-authz`** — auth-form spec gaps are security risks; tag both.

## False positives to avoid

- A controller without a request spec is fine if it has thorough service-layer coverage of the same logic. Check the layer below before flagging.
- `allow_any_instance_of` is sometimes the only sane option for legacy code; don't flag every instance, flag concentrations.
- Low coverage on `app/admin/` (ActiveAdmin) is conventional and not a defect.

## Reference points

Real-world Rails projects worth studying for this dimension. Patterns, not commandments — verify against your stack.

- **[`rspec/rspec-rails`](https://github.com/rspec/rspec-rails)** — the canonical RSpec-Rails project; its own `spec/` is a model of organization (per-layer subdirs, shared examples in `spec/support/`).
- **[`discourse/discourse`](https://github.com/discourse/discourse)** — large Rails app with disciplined factory use, parallelized CI, and explicit time-freezing. Look at `spec/spec_helper.rb` for WebMock setup.
- **[`thoughtbot/factory_bot_rails`](https://github.com/thoughtbot/factory_bot_rails)** — for non-cascading factory patterns.
- **[`mastodon/mastodon`](https://github.com/mastodon/mastodon)** — `spec/lib/` and `spec/services/` show service-layer coverage discipline.

_Verify cited paths on a recent commit before relying — Rails project structure shifts._
