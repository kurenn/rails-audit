# Security-and-authz revalidation

A focused second pass on every finding tagged `security-and-authz` (in `primary_dimension` OR `secondary_dimensions[]`). Mirrors the v0.2 money-revalidation pattern. Catches what general-purpose synthesis misses: timing-attack patterns, IDOR shapes, trusted-header identity, SQL interpolation, open redirects, SSRF.

Runs as **Step 4.6** in `SKILL.md` — *after* money-revalidation (4.5), *before* self-check (5.5).

---

## Why a second pass

Cluster D in the v0.2 fan-out covers security + AuthZ + money + governance simultaneously, time-bounded. Security-specific patterns get under-checked when the agent is racing to cover SSRF, JWT, IDOR, command injection, mass assignment, and token comparison in one budget.

A focused pass with **just** security-and-authz in scope catches things the cluster missed:

- Token comparisons using `==` against env vars, ENV[], session tokens, headers
- IDOR — `Resource.find(params[:id])` outside admin contexts without `current_user.<association>` scoping or Pundit policy
- Trusted-header identity — `request.headers['x-user-*']` used for auth without JWT verification
- SQL interpolation in `where()`, `find_by_sql`, `order(params[:..])`
- Open redirects — `redirect_to params[..]` without allowlist
- SSRF — outbound `Faraday.new(user_input)` / `URI.open(user_input)` without private-IP block
- Mass assignment — `params.permit!` / `to_unsafe_h` / `update(params)` shapes
- Command injection — backticks, `system`, `Open3.popen` with interpolated user input

---

## Scope

```
revalidation_set = findings.select { |f|
  f.primary_dimension == "security-and-authz" ||
  f.secondary_dimensions.include?("security-and-authz")
}
```

Plus a **proactive sweep**:

- `app/controllers/**/*authenticate*.rb`, `app/controllers/api/**/*.rb` (auth tier files)
- `app/controllers/webhooks/**/*.rb` (signature verification)
- Any file containing `Open3`, `system(`, `` ` `` (backtick), `eval`, `instance_eval`
- `app/models/**/*.rb` filtered for `validates :url`, `validates :host`, `validates :redirect_*` (URL validation regexes)

New findings discovered get `tool_origin: "security-revalidation"`.

---

## Statuses

For each finding in scope, set `findings[<id>].security_revalidation`:

| Status | When |
|---|---|
| `confirmed` | Finding stands as written; evidence verified against security checks |
| `refined` | Same finding, evidence/explanation/fix_sketch improved with security-specific detail |
| `rejected` | False positive on closer look (sets `ignored: true`, records `notes`) |
| `promoted` | Severity bumped one tier (medium → high → blocker) because a security-specific check applies |

`notes` is required for `refined`, `rejected`, `promoted`.

---

## The re-validation prompt

Same shape as money revalidation:

> You are re-validating a Rails-audit finding against security-and-authz-specific checks. The finding is below. The full security-and-authz dimension file is provided. The cited file's relevant region is provided.
>
> Decide: **confirmed** / **refined** / **rejected** / **promoted**.
>
> Output JSON only:
> ```json
> {
>   "id": "<finding id>",
>   "security_revalidation": {
>     "status": "...",
>     "notes": "..."
>   },
>   "updated_fields": { ... }
> }
> ```

The skill applies `updated_fields` after the response. May not change `id`, `primary_dimension`, `secondary_dimensions[]`.

---

## The six security re-checks

### S-RV-1 — Token comparison

Every comparison against an env var, ENV[], session token, or header used as a secret must use `ActiveSupport::SecurityUtils.secure_compare` (with length-equality guard) — not `==`. Influapp's `application_token == extracted_token` and coba's `params[:access_token] == ENV['TAX_FILE_ACCESS_TOKEN']` are canonical examples.

- **Confirmed** if the finding cites `==` and the fix correctly recommends `secure_compare`.
- **Promoted** medium → high if the original tagged it medium (timing attacks compound; auth comparisons are high-rubric default).
- **Refined** to add the length-equality guard reminder when missing.

### S-RV-2 — IDOR (broken object-level authorization)

Every `Resource.find(params[:id])` (or `find_by(id: params[:id])`) outside an admin context must:
- Be scoped to `current_user.<association>.find(...)`, OR
- Be guarded by a Pundit/CanCan policy on the action.

Backoffice/admin contexts may use unscoped `find` if a base controller (`AdminController` etc.) enforces `current_user.admin?` via a `before_action` or the controller is mounted behind an admin-only auth wall.

- **Confirmed** if the finding cites the unscoped find and the fix recommends scoping or policy.
- **Promoted** to blocker if the affected resource is user-owned and contains money/PII (e.g. `Payout.find(params[:id])` without ownership check).
- **Rejected** if closer inspection shows the controller IS behind an admin auth wall and the action only mutates admin-visible state.

### S-RV-3 — Trusted-header identity

Any `request.headers['x-user-*']`, `request.headers['x-account-*']`, or similar value used to look up the current user must be JWT-verified (or HMAC-verified) before trust. Influapp's `User.find_by(identity_platform_id: user_header)` was the canonical bad case.

- **Promoted to blocker** if the finding doesn't already note the JWT-verification gap (account takeover risk).
- **Confirmed** if blocker + cites JWKS verification fix.

### S-RV-4 — SQL interpolation

`where("col = '#{...}'")`, `find_by_sql("... #{...}")`, `order(params[:sort])`, `pluck(params[:col])` — any user input flowing into raw SQL is **blocker** unless the column/table set is explicitly allowlisted.

- **Promoted** any finding from medium/high to blocker.
- **Rejected** if the apparent interpolation is actually parameterized (e.g., `where("col = ?", value)` with `?` placeholder + bound parameter).

### S-RV-5 — Open redirect

`redirect_to params[:return_to]`, `redirect_to params[:next]`, etc. without a host allowlist is **high** (blocker if the post-login flow is affected — phishing chain).

- **Promoted** medium → high (phishing-vector default per rubric).
- **Refined** to recommend `URI.parse(target).host.in?(ALLOWED_HOSTS)` check.

### S-RV-6 — SSRF

Outbound `Faraday.new(user_input)`, `URI.open(user_input)`, `Net::HTTP.get(URI(user_input))` where the host is user-controlled and the request fires from server-side: **high** unless private-IP block + DNS pinning.

- **Promoted** if the finding doesn't note SSRF-specific mitigation (block 169.254/16, 10/8, 192.168/16, 127/8, ::1).
- **Confirmed** if the fix sketch already cites the IP-block list.

---

## What re-validation deliberately does *not* do

- **Does not re-run agent fan-out.** Focused pass on existing findings.
- **Does not modify primary or secondary dimensions.** Tagging is set in synthesis (Step 4).
- **Does not invoke external services.** All checks operate on source code.
- **Does not duplicate brakeman.** If brakeman already flagged the issue with high confidence, revalidation just confirms; if low confidence, revalidation may upgrade to refined/promoted.

---

## Render

`output-template.md` already conditionally appends `_Re-validated: <status>_` for findings with `money_revalidation`. v0.4 extends this to also render security_revalidation in the same shape (badge format: `_Sec-revalidated: <status>_` to distinguish).

---

## Future work (v0.5+)

- **Authorization revalidation** as a separate dimension (Pundit/CanCan policy coverage, role/permission checks).
- **Cross-finding consistency**: if S-RV-1 found a `==` token compare in `controller A`, check for the same pattern in `controller B`.
- **External integration**: optionally invoke `bundler-audit` and `brakeman --confidence 1` in a sub-step and merge results with revalidation outputs.
