# Dimension: Data integrity

DB-level constraints, validation duplication, soft-delete consistency, and transactional boundaries.

## What "good" looks like

- Every FK constrained at DB level (`add_foreign_key`).
- `NOT NULL` on every column that should never be null — model validation alone is not enough.
- Unique constraints at DB level for everything that has a unique scope at app level.
- Enums use either DB-level CHECK constraints (Postgres `CHECK`/native enums) or `ActiveRecord::Enum` with explicit mapping.
- `acts_as_paranoid` / `discard` use is consistent — every query in the affected models either explicitly opts into deleted rows (`with_deleted`) or relies on the default scope.
- Transactional boundaries are explicit on multi-step writes.
- No `find_or_create_by` without a corresponding unique index.
- Migrations are reversible (`up` + `down` or `change` with reversible commands only).

## Checks

### FK constraints

```bash
# Look for has_many/belongs_to and cross-reference with add_foreign_key in schema
grep -rn "belongs_to\b" app/models/ --include="*.rb" | wc -l
grep "add_foreign_key" db/schema.rb | wc -l
# Count discrepancy
```

Read `db/schema.rb` and list every `t.references` / `bigint` ending in `_id` that doesn't have a corresponding `add_foreign_key`. Each → **Medium** (or **High** for money/identity tables).

### NOT NULL discipline

```bash
# Columns without null: false
grep -E "t\.(string|text|integer|bigint|decimal|datetime|boolean)" db/schema.rb | grep -v "null: false"
```

Cross-reference with model validations:

```bash
grep -rn "validates_presence_of\|validates.*presence: true" app/models/ --include="*.rb"
```

Columns the model validates as present but the DB allows null → **Medium** (race condition allows null rows).

### Unique constraints

```bash
# Model-level
grep -rn "validates.*uniqueness" app/models/ --include="*.rb"
# DB-level
grep -E "add_index.*unique: true|index.*unique: true" db/schema.rb
```

Any `validates_uniqueness_of` without a corresponding DB unique index → **High** (concurrent inserts will both pass model validation, then both succeed in DB).

### Enums

```bash
grep -rn "enum :" app/models/ --include="*.rb"
```

`ActiveRecord::Enum` should:
- Use explicit hash mapping (`enum status: { pending: 0, paid: 1 }`), not array (which silently re-numbers on insertion).
- Have a DB-level CHECK constraint or a `validates_inclusion_of` matching.

Array-form enum → **High** (renaming/reordering breaks data silently).

### Soft delete consistency

```bash
# Models using paranoia / discard
grep -rn "acts_as_paranoid\|include Discard" app/models/ --include="*.rb"
# Queries that might forget about deleted scope
grep -rn "\.with_deleted\|\.only_deleted\|kept\|discarded" app/ --include="*.rb"
```

For each soft-deleted model, audit the controllers/services that query it. Does each query treat deleted rows correctly? Common bug: serializing a non-deleted record that `belongs_to` a deleted one and accessing the deleted association.

### Transactional boundaries

```bash
# Direct grep
grep -rn "ActiveRecord::Base.transaction\|self.class.transaction\|transaction do" app/ --include="*.rb"
# Find multi-write methods that should but don't use one
```

Multi-step writes (e.g. "create payout, then create ledger entry, then update user balance") need an explicit transaction. Manual read of money services + multi-write controllers.

### `find_or_create_by` without unique index

```bash
grep -rn "find_or_create_by\|first_or_create" app/ --include="*.rb"
```

Each match → cross-reference with `db/schema.rb`. No corresponding unique index → **High** (race window creates duplicates).

### Migrations

```bash
ls -t db/migrate/*.rb 2>/dev/null | head -10
```

For each, check:
- Reversible? (`change` method or both `up`+`down`)
- Data backfill in same migration as schema change? (separate them)
- `add_index` on a large table without `algorithm: :concurrently`?
- `add_column ..., null: false` without `default:` on a non-empty table?

### Schema vs model drift

```bash
# Models that reference columns no longer in schema
ruby -e '
schema_cols = {}
File.read("db/schema.rb").scan(/create_table[^"]*"(\w+)"[^d]*do \|t\|(.*?)end/m) { |table, body| schema_cols[table] = body.scan(/t\.\w+\s+:(\w+)/).flatten }
# Then for each app/models/*.rb file, look for hardcoded attribute names that arent in schema
' 2>/dev/null
```

(This is hard to fully automate; light pass: cross-reference any `validates :col_name` against the schema columns for that model.)

### Counter cache integrity

```bash
grep -rn "counter_cache" app/models/ --include="*.rb"
```

Counter caches drift over time. Any model with a `counter_cache: true` association needs a periodic reconciliation rake task. Absent → **Low**.

### State machines

```bash
grep -E "gem ['\"](aasm|state_machines|statesman|workflow)['\"]" Gemfile
grep -rn "include AASM\|state_machine" app/models/ --include="*.rb"
```

For models with state, prefer an explicit state machine over a `status` string with hand-rolled transitions. Hand-rolled state in a money/auth model → **Medium**.

### `acts_as_paranoid` quirks

- `belongs_to` with `with_deleted: true` is correct for soft-delete-aware joins, but means the parent might be a deleted record. Surface this.
- `dependent: :destroy` on a soft-deleted parent silently soft-deletes children, which is usually fine, but `dependent: :delete_all` *hard* deletes — flag if mixed.

## Severity calibration

| Pattern | Default tier |
|---|---|
| `validates_uniqueness_of` without DB unique index | High |
| `find_or_create_by` without unique index | High |
| FK without DB-level `add_foreign_key` (money/identity tables) | High |
| FK without DB-level constraint (other tables) | Medium |
| Model `presence: true` but column allows null | Medium |
| Array-form enum | High |
| Hand-rolled state on money/auth model | Medium |
| Soft-delete model with inconsistent `with_deleted` use in queries | Medium (High if data leakage possible) |
| Multi-step money write without explicit transaction | High |
| Migration `add_column null: false` without `default:` on large table | High |
| Migration `add_index` without `concurrently:` on large table | High |
| Counter cache without reconcile task | Low |
