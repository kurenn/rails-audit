# Dimension: Data governance

PII inventory, encryption at rest, audit logs, data retention, right-to-be-forgotten. Sometimes a blocker (regulated industries), sometimes hygiene (early-stage B2C).

## What "good" looks like

- A documented PII inventory: which columns are PII, where they appear in logs, exports, backups.
- Sensitive columns encrypted with `ActiveRecord::Encryption` (Rails 7+) or `attr_encrypted`.
- Audit log of mutations on sensitive records (paper_trail, audited).
- Right-to-be-forgotten path: a request flow that hard-deletes (not soft-deletes) PII.
- Data retention policies: logs ≤ N days, backups ≤ N days, deleted-user data purged after N days.
- Backup encryption (managed by the DB host, but verified).
- Database connection encrypted (`sslmode: require` or stricter for managed Postgres).
- Subprocessor list current if app has data processors (Stripe, Twilio, GCS, etc.).

## Checks

### PII inventory

This is a manual read. For each model, list which columns contain PII per common categories:

| Category | Examples |
|---|---|
| Direct identifiers | name, email, phone, government IDs, identity_platform_id |
| Quasi-identifiers | address, DOB, IP, user-agent |
| Financial | bank account, card last4, Stripe customer ID, payout details |
| Authentication | password (hashed), API tokens, OAuth tokens, MFA secrets |
| Behavioral | location history, search history, message content |

```bash
# Heuristic surface
grep -rnE "(email|phone|address|ssn|tax_id|dob|birthday|passport|password|token|api_key|stripe_)" db/schema.rb | head -50
```

Document the inventory in the report appendix. The inventory is the deliverable here, not a count.

### Encryption at rest

```bash
# Rails 7 native
grep -rn "encrypts\b" app/models/ --include="*.rb"
# attr_encrypted gem
grep -rn "attr_encrypted\|attr_encrypted_options" app/models/ --include="*.rb"
# lockbox
grep -E "gem ['\"]lockbox['\"]" Gemfile
```

Sensitive columns *not* encrypted at app level → **Medium** (relies on DB-level encryption alone). Required by GDPR/HIPAA contexts → **Blocker** if missing.

### Database connection encryption

```bash
grep -E "sslmode|ssl: true" config/database.yml
```

Production `sslmode` of `disable` / `allow` / `prefer` → **High**. Should be `require`/`verify-ca`/`verify-full`.

### Audit log

```bash
grep -E "gem ['\"](audited|paper_trail|logidze)['\"]" Gemfile
ls db/migrate/ 2>/dev/null | grep -i "audit\|version"
```

Money/auth models without audit gem → **High** in compliance contexts (SOC2, HIPAA), **Medium** otherwise.

### Right-to-be-forgotten

Look for an explicit deletion endpoint or rake task that hard-deletes PII for a user:

```bash
grep -rnE "DELETE.*users?\b|destroy.*user\b" app/controllers/ app/services/ --include="*.rb"
grep -rn "purge_pii\|forget_user\|gdpr\|data_export" app/ lib/ --include="*.rb"
```

Soft-delete-only on a user with PII → **High** in EU/regulated contexts (GDPR Article 17).

The check on `acts_as_paranoid` is tricky here: deleted users still have all their PII in the DB, just hidden. A real "delete" endpoint must hard-delete or null out PII columns.

### Data export (right to portability)

```bash
grep -rn "data_export\|user_export\|export_user" app/ lib/ --include="*.rb"
```

Required by GDPR; nice-to-have otherwise. Absent → **Medium** in EU contexts.

### Retention policies

This is a config + ops question, not just code. Surface:
- Log retention: where are logs stored? For how long? (Cloud Run's default Stackdriver retention, Heroku Papertrail/Logentries, etc.)
- Backup retention: managed DB host config — verify with user, document.
- Soft-deleted record retention: if `acts_as_paranoid` keeps rows forever, that's a retention violation.

```bash
# Soft-delete cleanup
grep -rn "really_destroy_all\|purge_deleted\|cleanup\|housekeeping" app/jobs/ app/services/ lib/tasks/ 2>/dev/null
```

No mechanism to ever purge soft-deleted records → **Medium** in compliance contexts.

### Subprocessor list

If the app has external data processors (Stripe, Twilio, GCS, identity providers, analytics), there should be an internal list (and sometimes a public one for B2B compliance).

This is less a code check and more a documentation check — surface as "verify with user" if no obvious subprocessor doc exists in the repo.

### Cookie / consent (HTML apps only)

API-only apps skip this. HTML apps with EU users need:
- Cookie banner (legal, but also a code check — is one rendered?)
- Differentiation between necessary, analytics, marketing cookies.

### Logging discipline

PII in logs is a governance concern. Cross-reference with `dimensions/observability.md`:
- `filter_parameters` includes all sensitive fields.
- Custom log lines (e.g. `Rails.logger.info "User #{user.email} did X"`) are flagged → log the ID, not the email.

```bash
# Custom log lines with PII
grep -rnE "Rails\.logger\..*(email|phone|password|token|address|ssn|dob)\b" app/ --include="*.rb" | head
```

Each → **Medium** (or **High** for password/token leaking into logs).

### Service account least privilege

(Cross-listed with security.) Cloud service accounts should follow least privilege — one SA with `Storage Admin` + `Cloud Tasks Admin` + `Identity Toolkit Admin` is overprivileged.

This is rarely visible in the repo; surface as "verify with user" unless IAM config is in repo (Terraform / Pulumi).

## Cross-cuts

- **`security-and-authz`** — encryption at rest, PII in logs, secrets handling cross both.
- **`observability`** — PII in custom log lines and error tracker payloads cross with observability.
- **`data-integrity`** — soft-delete on PII tables, retention policies cross with integrity.
- **`money-and-payments`** — audit trail on money mutations is governance primary; money secondary.
- **`deploy-and-ci`** — prod-points-at-staging or unencrypted secrets in workflows are deploy *and* governance.

## Severity calibration

| Pattern | Default tier |
|---|---|
| Sensitive columns unencrypted (compliance context) | Blocker |
| Sensitive columns unencrypted (no compliance) | Medium |
| `sslmode: disable/allow/prefer` for managed Postgres | High |
| No hard-delete path for users (EU users) | High |
| No audit log on money/auth models (compliance) | High |
| No audit log on money/auth models (no compliance) | Medium |
| PII in custom log lines | Medium (High for password/token) |
| No retention policy for soft-deleted records | Medium |
| No data export endpoint (EU users) | Medium |
| Overprivileged cloud service account | Medium |
| No documented subprocessor list (B2B) | Low |
