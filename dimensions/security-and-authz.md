# Dimensions: Security + Authorization

OWASP Top 10 mapped to Rails patterns. Authorization separated from authentication because most Rails breaches are AuthZ, not AuthN.

## What "good" looks like

- AuthN strategy is clear (Devise / JWT / Warden / custom) and one strategy is consistently applied.
- AuthZ is a separate layer (Pundit / CanCanCan / native policies). Every controller action that touches a user-owned resource scopes the lookup to `current_user`. Policies have specs.
- Token comparison via `ActiveSupport::SecurityUtils.secure_compare`.
- Webhooks: signature verified with `Stripe::Webhook.construct_event` (or equivalent), event ID persisted to a dedup table, replay window enforced.
- Strong params everywhere; no `.permit!` / `.permit(:*)` / `params.to_unsafe_h` on user input.
- No SQL string interpolation. `where("col = ?", val)` form everywhere.
- No `redirect_to params[:return_to]` without an allowlist.
- No `Marshal.load` / `YAML.load` / `JSON.parse(..., create_additions: true)` on user input.
- Outbound HTTP from user-supplied URLs goes through an SSRF-safe fetcher (private IP block).
- Security headers set: HSTS, CSP, X-Frame-Options, X-Content-Type-Options, Referrer-Policy.
- Rate limiting on login, password reset, signup, expensive endpoints.
- PII parameters in `Rails.application.config.filter_parameters`.

## Checks

### Authentication

```bash
# Strategy
grep -E "gem ['\"](devise|warden|jwt|clearance|doorkeeper|sorcery)['\"]" Gemfile
# Token comparison
grep -rn "==.*token\|token.*==" app/controllers/ --include="*.rb" | grep -v "secure_compare"
# JWT pitfalls
grep -rn "JWT\.\(decode\|encode\)" app/ --include="*.rb"
grep -rn "algorithm" app/ --include="*.rb" | grep -i "jwt\|token"
# Look for alg: 'none' acceptance
grep -rn "algorithm.*none\|alg.*none" app/ --include="*.rb"
# Identity headers trusted without verification (the influapp pattern)
grep -rn "request.headers\[" app/controllers/ --include="*.rb" | grep -iE "user|identity|sub"
```

Manual reads:
- `app/controllers/application_controller.rb`
- `app/controllers/api/authenticated_controller.rb` (or equivalent base)
- Any concern under `app/controllers/concerns/` involved in auth

### Authorization

```bash
# Policy framework presence
grep -E "gem ['\"](pundit|cancancan|action_policy)['\"]" Gemfile
# Policy spec presence
ls spec/policies/ 2>/dev/null
# IDOR pattern: find_by(id: params[:id]) or .find(params[:id]) without current_user scope
grep -rn "\.find(params\[:id\])\|\.find_by(id: params\[:id\])" app/controllers/ --include="*.rb" | head -30
# These need eyeballing — sometimes scoped via before_action, sometimes not
```

For every match above, check whether the surrounding controller action scopes via `current_user.<association>.find(...)` or runs through a policy. Unscoped finds on user-owned resources → **Blocker**.

### Strong params

```bash
grep -rn "\.permit!\|\.to_unsafe_h\|\.permit(:\*)" app/ --include="*.rb"
grep -rn "params\.\(to_unsafe_hash\|permit!\)" app/ --include="*.rb"
```

Any match → **High** unless in an explicitly admin-only context.

### SQL injection

```bash
# Interpolation in where/find_by_sql/order
grep -rnE 'where\(\s*["\x27].*#\{' app/ --include="*.rb"
grep -rn "find_by_sql" app/ --include="*.rb"
grep -rnE 'order\(.*params\[' app/ --include="*.rb"
grep -rnE 'pluck\(.*#\{' app/ --include="*.rb"
```

Any positive match → **Blocker** until proven safe.

### XSS / output safety

```bash
# Views
grep -rn "html_safe\|raw(\|<%=raw\b" app/views/ 2>/dev/null
# Mailers
grep -rn "html_safe\|raw(" app/mailers/ app/views/*_mailer/ 2>/dev/null
```

Each `html_safe` / `raw` call needs a justification that the input is server-controlled. Concentrations (more than a handful) → **High**.

### CSRF (non-API routes)

```bash
grep -rn "skip_before_action.*verify_authenticity_token\|protect_from_forgery.*null_session\|protect_from_forgery.*:none" app/controllers/ --include="*.rb"
```

`skip_before_action :verify_authenticity_token` on non-webhook controllers → **High**.

### Webhook security

```bash
ls app/controllers/webhooks/ 2>/dev/null
# For each webhook controller:
# - signature verification present?
# - event ID dedup (DB table or cache)?
# - timestamp/replay window?
grep -rn "Stripe::Webhook\|Webhook.construct_event\|signature\|HTTP_X_.*_SIGNATURE" app/controllers/webhooks/ --include="*.rb"
# Look for an Events / processed_events table for dedup
grep -rn "processed_event\|stripe_event\|webhook_event" app/models/ db/schema.rb 2>/dev/null
```

- No signature verification → **Blocker**.
- Signature verified but no event-ID dedup → **High** (Stripe retries on its own).
- Comparison with `==` instead of `secure_compare` for HMAC → **High**.

### SSRF

```bash
# Outbound HTTP with user input
grep -rnE "Faraday|HTTParty|RestClient|Net::HTTP|URI\.\(open\|parse\)" app/ --include="*.rb" | head
# Then for each match, check whether the URL is user-derived
```

Look for image proxies, link unfurls, OAuth redirect_uri flows, webhook outbound. Any user-controlled URL fetch without an allowlist + private-IP block → **High**.

### Deserialization

```bash
grep -rn "Marshal\.load\|YAML\.load\b\|JSON\.parse.*create_additions" app/ lib/ --include="*.rb"
```

`YAML.load` (vs `YAML.safe_load`) on user input → **Blocker**. `Marshal.load` on user input → **Blocker**.

### Command injection

```bash
grep -rnE "(system|exec|`|%x|spawn|Open3\.(popen|capture))" app/ lib/ --include="*.rb" | grep -E "\#\{|params|input" | head
```

Any shell call with interpolated user input → **Blocker**.

### Open redirect

```bash
grep -rn "redirect_to params\|redirect_to.*\.params" app/controllers/ --include="*.rb"
```

`redirect_to params[:return_to]` or similar without an allowlist → **High**.

### Path traversal / file uploads

```bash
grep -rnE "File\.(open|read|write).*params" app/ lib/ --include="*.rb"
grep -rnE "Zip::\w+\.open" app/ lib/ --include="*.rb"  # ZIP slip
# ActiveStorage validations?
grep -rn "validates.*content_type\|active_storage_validations" app/models/ Gemfile
```

User-controlled file path → **Blocker**. ZIP extraction without sanitization → **Blocker**. ActiveStorage attachments without content-type validation → **Medium**.

### Headers

```bash
# secure_headers gem or manual setup
grep -E "gem ['\"]secure_headers['\"]" Gemfile
grep -rn "default_headers\|content_security_policy" config/ app/controllers/application_controller.rb
```

No CSP and no `secure_headers` gem on a non-API app → **Medium**. API-only with no CORS hardening → check `rack-cors` config.

### Rate limiting

```bash
grep -E "gem ['\"]rack-attack['\"]" Gemfile
test -f config/initializers/rack_attack.rb && cat config/initializers/rack_attack.rb
```

No rate limit on login/password-reset → **High**.

### PII in logs

```bash
grep -rn "filter_parameters" config/ app/
# Standard list should include :password, :token, :secret, :api_key, plus app-specific PII
```

If `password`/`token`/`api_key` not in `filter_parameters` → **High**.

### Account enumeration

Read login/password-reset/signup endpoints. The error message and HTTP status for "email exists" must not differ from "email does not exist". Any difference → **Medium** (or **High** for password reset).

### Mass assignment

```bash
grep -rn "attr_accessible\|attr_protected" app/ --include="*.rb"  # Rails 3 era — should be gone
grep -rnE "\.assign_attributes\b|\.update\(\s*params\s*\)" app/controllers/ --include="*.rb"
```

`update(params)` directly without `.permit` → **Blocker**. `assign_attributes` with raw params → **High**.

### Race conditions

```bash
# find_or_create_by without unique index — needs schema cross-reference
grep -rn "find_or_create_by\|first_or_create" app/ --include="*.rb"
```

Each match needs a corresponding unique DB index — verify in `db/schema.rb`.

## Severity calibration

| Pattern | Default tier |
|---|---|
| Token compared with `==` | High (Blocker if it gates payments/admin) |
| Identity header trusted without JWT verification | Blocker |
| IDOR — `Resource.find(params[:id])` without owner scoping | Blocker |
| SQL string interpolation | Blocker |
| `Marshal.load` / `YAML.load` on user input | Blocker |
| Webhook without signature verification | Blocker |
| Webhook with signature but no event-ID dedup | High |
| `redirect_to params[...]` without allowlist | High |
| User-controlled URL fetch without SSRF protection | High |
| `permit!` / `to_unsafe_h` on user input | High |
| `skip_before_action :verify_authenticity_token` outside webhooks | High |
| Account enumeration on password reset | High |
| No rate limit on login | High |
| No CSP / `secure_headers` on HTML-rendering app | Medium |
| `html_safe` / `raw` concentration | Medium |
| `filter_parameters` doesn't include token/password | High |
