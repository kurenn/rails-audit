# Cost estimation

The skill produces a pre-flight estimate of token usage before fanning out agents in Step 3. Users can sanity-check the estimate, set a `--budget=<tokens>` cap, or abort. Estimates land in `cost.estimated_input_tokens` and `cost.estimated_output_tokens`; actuals (when available) land in `cost.actual_*`.

This is **not** a hard contract — token usage varies with project structure, agent verbosity, and tool output size. The heuristic is calibrated to be within ~30% of actual over standard-mode runs of typical Rails projects (~10K–50K LOC). It will be wrong on outliers (huge monorepos, unusual layouts).

---

## Heuristic (v0.3 — calibrated against influapp + coba dogfoods)

```
mode_multiplier = {
  "quick":    0.30,
  "standard": 1.00,
  "deep":     1.50
}[audit.mode]

base_input    = 5_000
base_output   = 3_000
per_dim_in    = 2_000
per_dim_out   =   600
per_kloc_in   =   300   # v0.3 — bumped from 200 (influapp v0.1 underestimate)
per_kloc_out  =    10
per_kloc_lib  =   100   # NEW v0.3 — non-app/ code factor
per_kloc_eng  =    50   # NEW v0.3 — engines/* factor (typically 0)

dims    = audit.scope.length          # if "all", expand to 18
loc_app = total LOC under app/
loc_lib = total LOC under lib/         # NEW v0.3
loc_eng = total LOC under engines/     # NEW v0.3 — most projects: 0

estimated_input  = mode_multiplier * (base_input  + per_dim_in  * dims
                                      + per_kloc_in  * (loc_app / 1000)
                                      + per_kloc_lib * (loc_lib / 1000)
                                      + per_kloc_eng * (loc_eng / 1000))
estimated_output = mode_multiplier * (base_output + per_dim_out * dims + per_kloc_out * (loc_app / 1000))
```

Round to the nearest 1,000 for display.

### Worked examples (v0.3 calibrated)

**influapp-api dogfood** (Rails 7, ~13K LOC under `app/`, ~2K under `lib/`, all 18 dimensions, standard mode):

```
estimated_input  = 1.0 * (5000 + 2000 * 18 + 300 * 13 + 100 * 2) = 5000 + 36000 + 3900 + 200 = 45_100
estimated_output = 1.0 * (3000 +  600 * 18 +  10 * 13)            = 3000 + 10800 + 130       = 13_930
```

≈ 45K input / 14K output. The v0.1 influapp run was ~52K / 14K — v0.3 estimate within ~13% MAPE (improvement from v0.2's ~16%).

**coba dogfood** (Rails 7.1, ~28K LOC under `app/`, ~5K under `lib/`, all 18 dims, standard):

```
estimated_input  = 1.0 * (5000 + 2000 * 18 + 300 * 28 + 100 * 5) = 5000 + 36000 + 8400 + 500 = 49_900
estimated_output = 1.0 * (3000 +  600 * 18 +  10 * 28)            = 3000 + 10800 + 280       = 14_080
```

≈ 50K input / 14K output. (v0.2 reported 56K under the old constants — the new constants give 50K, closer to actual when token accounting becomes available.)

**Money-only scoped audit on influapp** (`--only=money`, 1 dimension):

```
estimated_input  = 1.0 * (5000 + 2000 *  1 + 300 * 13 + 100 * 2) = 5000 + 2000 + 3900 + 200 = 11_100
estimated_output = 1.0 * (3000 +  600 *  1 +  10 * 13)            = 3000 + 600 + 130          = 3_730
```

≈ 11K / 4K — much cheaper, as expected.

**Quick mode on influapp** (static-only, no fan-out):

```
estimated_input  = 0.3 * 45_100 = 13_530
estimated_output = 0.3 * 13_930 =  4_179
```

≈ 14K / 4K.

---

## How LOC is measured

Use:

```bash
loc_app = $(find app -name "*.rb" -exec wc -l {} + 2>/dev/null | tail -1 | awk '{print $1}')
```

Falls back to `0` if `app/` is empty (rare for Rails). v0.3 added `loc_lib` and `loc_engines` factors based on coba dogfood (substantial Monex bindings under `lib/`).

```bash
loc_lib = $(find lib -name "*.rb" -exec wc -l {} + 2>/dev/null | tail -1 | awk '{print $1}' || echo 0)
loc_eng = $(find engines -name "*.rb" -exec wc -l {} + 2>/dev/null | tail -1 | awk '{print $1}' || echo 0)
```

---

## When to prompt the user

Run prompt **P5** in `prompts.md` when *both* of these are true:

1. `--budget` is **not** explicitly set, AND
2. `estimated_input_tokens > 30_000`

Below 30K input, the audit is cheap enough that asking is just friction. Above 30K, surface the estimate.

The prompt accepts:

- `Y` (default): proceed without budget cap.
- `n`: abort.
- `budget=N` (or numeric input alone): set `--budget=N` and proceed.

---

## Budget enforcement

When `--budget=<N>` is set:

1. Track running token usage (input + output) across agent calls and Bash invocations.
2. **Before each agent call**, compute `remaining = budget - usage_so_far`. If the agent's expected cost (using `per_dim_in + per_dim_out` as a conservative upper bound) would exceed `remaining`, abort that agent call.
3. When the budget is hit:
   - Save the **partial JSON** to `tmp/rails-audit/report-YYYY-MM-DD.json` with `audit.finished_at` set to the abort time.
   - Note in `summary.verdict`: `"Audit aborted at <usage>/<budget> tokens; partial report only — <N> dimensions skipped."`
   - Render a **partial markdown report** that includes the dimensions that completed; flag missing dimensions explicitly.
4. **Step 7 (write outputs)** still runs — the user gets whatever was completed.

Hard floor: budget < 5,000 tokens is rejected at parse time (no audit can fit).

---

## Cost appendix in the report

The schema already supports `cost{}` from PR#1. Populated fields:

```json
"cost": {
  "estimated_input_tokens":  43600,
  "estimated_output_tokens": 13930,
  "actual_input_tokens":     null,
  "actual_output_tokens":    null,
  "budget_tokens":           null
}
```

`actual_*` is populated when the runtime exposes it (via the harness or per-agent accounting). When unavailable, leave `null` and the markdown render shows `—`.

---

## Calibration plan

After running the v0.2 skill against ~5 real Rails projects:

1. Record each project's actual input/output token usage alongside the heuristic's estimate.
2. Compute mean absolute percentage error (MAPE) per mode.
3. Adjust the constants in v0.3 if MAPE > 30% on standard mode.
4. Add `loc_lib` and `loc_engines` factors if outliers correlate with non-`app/` code volume.

---

## What this dimension deliberately does *not* do

- **Does not bill or charge.** It estimates and records. Billing is between the user and the LLM provider.
- **Does not enforce per-agent budgets** (only the global budget). Per-agent caps are a v0.3 idea if needed.
- **Does not retry within budget.** If a tool fails and a retry would exceed the budget, the failure stays.
- **Does not include tool-output ingestion in input estimates separately.** Tool outputs (Brakeman JSON, Bundler-audit text, etc.) are folded into the `per_kloc_in` term as an implicit factor.
