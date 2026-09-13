# PostgreSQL & pg-promise — Technical Standardization (Practical, "Right vs Wrong")

This guide is written to be enforceable in code reviews. It covers schema conventions, data types, integrity constraints, indexing, and query patterns for PostgreSQL accessed through `pg-promise` from persistence adapters.

Applies when: writing a migration, implementing or reviewing a repository (persistence adapter), designing a data model, or optimizing SQL in review.

See also: `general-guidelines.md` for naming conventions, `api-style-guide.md` for the pagination contract this schema must support.

---

## 1) Naming Conventions

Everything in **English**, columns in **snake_case**, tables in **plural**.

✅ **Right**

```sql
CREATE TABLE customer_invoices (
    id uuid PRIMARY KEY,
    tenant_id uuid NOT NULL,
    issued_at timestamptz NOT NULL
);
```

❌ **Wrong**

```sql
CREATE TABLE CustomerInvoice (   -- singular, PascalCase
    ID uuid,
    tenantID uuid,               -- camelCase
    dataEmissao timestamptz      -- not English
);
```

Unquoted identifiers are folded to lowercase by PostgreSQL. A quoted `"customerInvoice"` forces every subsequent query to quote it too — never create quoted mixed-case identifiers.

### Constraint and index names

Name every constraint explicitly. Auto-generated names are unpredictable and make migrations that drop them fragile.

| Object | Pattern | Example |
| --- | --- | --- |
| Primary key | `<table>_pk` | `users_pk` |
| Foreign key | `<table>_<ref>_fk` | `users_tenant_fk` |
| Unique | `<table>_<cols>_uk` | `users_tenant_email_uk` |
| Check | `<table>_<col>_ck` | `users_status_ck` |
| Index | `idx_<table>_<purpose>` | `idx_users_list_active` |

---

## 2) Data Types

| Domain | Type | Never |
| --- | --- | --- |
| Identifier | `uuid` | `serial`, `bigserial` on tenant-scoped tables |
| Text | `text` | `varchar(n)` as a validation mechanism |
| Money / quantity | `numeric(19,4)` | `float`, `double precision`, `real` |
| Timestamp | `timestamptz` | `timestamp` (without time zone) |
| Date only | `date` | `text` |
| Boolean | `boolean` | `smallint`, `char(1)` |
| Enumerated set | `text` + `CHECK` | native `ENUM` type |
| Schemaless payload | `jsonb` | `json`, `text` |

### Money is never floating point

```sql
-- ❌ Wrong: 0.1 + 0.2 <> 0.3
amount double precision NOT NULL

-- ✅ Right: exact decimal arithmetic
amount numeric(19,4) NOT NULL
```

Binary floating point cannot represent most decimal fractions. A rounding error of 0.0000001 per row becomes a reconciliation failure at scale.

### Always `timestamptz`, never `timestamp`

`timestamp` stores a wall-clock reading with no time zone, so the same value means a different instant depending on who reads it. `timestamptz` stores an absolute instant and converts on input/output. Store UTC, convert at the presentation edge.

### `text` over `varchar(n)`

In PostgreSQL there is no performance difference. A `varchar(n)` limit is a schema-level constraint that requires a migration to change and produces a database error instead of a validation message. Enforce length in the DTO (see `api-style-guide.md` §7); use `text` in the schema. Use `CHECK (length(col) <= n)` only when the limit is a genuine domain invariant.

### `CHECK` over native `ENUM`

Adding a value to a native `ENUM` is a DDL operation with locking implications, and removing one is effectively impossible. A `text` column with a `CHECK` constraint is altered with a simple constraint swap.

```sql
status text NOT NULL CONSTRAINT users_status_ck CHECK (status IN ('ACTIVE', 'INACTIVE', 'SUSPENDED'))
```

---

## 3) Primary Keys and Identifiers

**Primary key is `uuid`, generated in the application as UUIDv7.**

```sql
id uuid PRIMARY KEY
```

Rationale:

- **Generated in the use case, not the database.** The domain object is fully valid the moment it is constructed — no partially-initialized entity waiting for an INSERT to assign its identity. This also lets a single transaction build an object graph without round-tripping for each ID.
- **UUIDv7, not v4.** v7 embeds a millisecond timestamp in its high bits, so values are time-ordered. Random v4 keys scatter inserts across the whole B-tree, causing page splits and index bloat; v7 appends to the right edge like a sequential key while remaining globally unique and non-enumerable.
- **Not `serial`.** Sequential integers leak record counts, enable enumeration across tenants, and collide when merging data across environments or shards.

❌ **Wrong**

```sql
id uuid PRIMARY KEY DEFAULT gen_random_uuid()   -- v4: random, index-hostile
id bigserial PRIMARY KEY                        -- enumerable, leaks volume
```

---

## 4) Audit Columns

**Every table carries `created_at` and `updated_at` as `timestamptz NOT NULL`.**

```sql
created_at timestamptz NOT NULL DEFAULT now(),
updated_at timestamptz NOT NULL DEFAULT now()
```

`updated_at` must actually be updated. Either set it explicitly in every `UPDATE` statement in the repository, or install a trigger — pick one convention per project and apply it uniformly. A half-maintained `updated_at` is worse than none, because it is silently trusted.

```sql
CREATE OR REPLACE FUNCTION set_updated_at() RETURNS trigger AS $$
BEGIN
    NEW.updated_at = now();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER users_set_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```

---

## 5) Integrity Constraints

The database is the last line of defense for **atomic integrity**. Complex business rules live in the Domain; the schema guarantees what the application cannot guarantee under concurrency.

### Foreign keys are mandatory

Declare the referential action explicitly — the default `NO ACTION` is rarely what you mean.

```sql
CONSTRAINT users_tenant_fk FOREIGN KEY (tenant_id)
    REFERENCES tenants (id) ON DELETE RESTRICT
```

- `ON DELETE RESTRICT` — the default choice. Deleting a parent with children is a bug; surface it.
- `ON DELETE CASCADE` — only for true composition, where the child has no independent existence (e.g. `invoice_items` under `invoices`).
- `ON DELETE SET NULL` — only when the column is genuinely nullable and orphaning is a valid state.

**Always index the FK column.** PostgreSQL indexes the referenced side automatically but not the referencing side, so an un-indexed FK makes every parent delete or update a full scan of the child table.

### Unique constraints enforce invariants, not validation

A `SELECT ... WHERE email = $1` followed by an `INSERT` is a race: two concurrent requests both see "not found" and both insert. Only a unique index prevents this.

```sql
CONSTRAINT users_tenant_email_uk UNIQUE (tenant_id, email)
```

Note the tenant in the key — uniqueness is almost always scoped, and a globally unique `email` is usually a modeling error in a multi-tenant system.

With soft delete, use a partial unique index so deleted rows don't block reuse of the value:

```sql
CREATE UNIQUE INDEX users_tenant_email_uk ON users (tenant_id, email)
WHERE deleted_at IS NULL;
```

### `NOT NULL` by default

A column is nullable only when "unknown" or "not applicable" is a meaningful domain state. Nullable-by-default schemas push null handling into every query and every mapper.

---

## 6) Indexing

An index exists to serve a specific query. Every index added should be traceable to a query in a repository; every index that isn't is pure write overhead.

### Composite index column order

Order columns **equality first, then sort, then range**:

```sql
-- Query: WHERE tenant_id = $1 AND deleted_at IS NULL ORDER BY created_at DESC, id DESC
CREATE INDEX idx_users_list_active ON users (tenant_id, created_at DESC, id DESC)
WHERE deleted_at IS NULL;
```

`tenant_id` is equality → leftmost. `created_at, id` is the sort → next, with matching direction so the index is read in order rather than sorted afterward. A range predicate goes last, because everything after a range scan in the index cannot be used for further filtering.

### Partial indexes for soft delete

When nearly every query filters `deleted_at IS NULL`, a partial index excludes deleted rows from the index entirely — smaller, faster, and it keeps deleted rows from consuming cache.

### Verify with `EXPLAIN ANALYZE`

Do not assume an index is used. Before merging a new query or index:

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;
```

Look for `Seq Scan` on a large table, a `Sort` node that the index should have eliminated, and a row-estimate that diverges wildly from actual — that last one usually means stale statistics or a predicate the planner cannot estimate.

---

## 7) Query Standards

### `SELECT *` is prohibited

```sql
-- ❌ Wrong
SELECT * FROM users WHERE tenant_id = $1;

-- ✅ Right
SELECT u.id, u.email, u.status, u.created_at
FROM users u
WHERE u.tenant_id = $1;
```

`SELECT *` transfers columns nobody reads, breaks silently when a column is added or reordered, prevents index-only scans, and makes the query's actual dependencies invisible in review.

### Explicit joins only

```sql
-- ❌ Wrong: implicit join, condition buried in WHERE
SELECT u.id, t.name FROM users u, tenants t WHERE u.tenant_id = t.id AND u.status = 'ACTIVE';

-- ✅ Right: join condition separated from filtering
SELECT u.id, t.name
FROM users u
INNER JOIN tenants t ON t.id = u.tenant_id
WHERE u.status = 'ACTIVE';
```

Comma-joins make an accidentally-omitted condition a silent cartesian product instead of a syntax error.

### Alias every table; qualify every column

In any query touching more than one table, unqualified columns are ambiguous to readers and break when a joined table later gains a column of the same name.

### Deterministic `ORDER BY` for pagination

```sql
ORDER BY created_at DESC, id DESC
```

Sorting by a non-unique column alone gives no total order — rows with equal `created_at` can be returned in a different sequence on each query, so paginating skips and duplicates records. Always append a unique tiebreaker, and make the index match the sort direction.

### Prefer `EXISTS` over `COUNT` for existence checks

```sql
-- ❌ Wrong: counts every matching row
SELECT COUNT(*) FROM users WHERE tenant_id = $1 AND email = $2;

-- ✅ Right: stops at the first match
SELECT EXISTS(SELECT 1 FROM users WHERE tenant_id = $1 AND email = $2) AS exists;
```

### Avoid N+1 in repositories

Loading a list and then querying per item is the most common performance defect in adapters. Fetch related rows in one statement with a join, or batch with `WHERE id = ANY($1)`.

```typescript
// ✅ Right: one round trip for the whole batch
await this.db.any('SELECT id, name FROM tenants WHERE id = ANY($1)', [tenantIds]);
```

---

## 8) Security: Parameterized Queries Only

**Never interpolate values into SQL.** Use `pg-promise` placeholders.

✅ **Right**

```typescript
await this.db.any('SELECT id FROM users WHERE tenant_id = $1 AND status = $2', [tenantId, status]);
```

❌ **Wrong**

```typescript
await this.db.any(`SELECT id FROM users WHERE tenant_id = '${tenantId}'`);
await this.db.any('SELECT id FROM users WHERE status = $1', [`'${status}'`]);
```

This holds even when the value "comes from an enum" or "is already validated" — validation drifts, the query survives, and the injection point is now invisible.

For dynamic **identifiers** (table or column names, which cannot be parameterized), use `pg-promise`'s `pgp.as.name()` escaping and validate against an allowlist of known columns. Never build an `ORDER BY` directly from a query parameter.

```typescript
const SORTABLE = { created_at: 'created_at', email: 'email' } as const;
const column = SORTABLE[query.sortBy] ?? 'created_at';   // allowlist, not interpolation
```

---

## 9) pg-promise Usage

### Pick the right result method

| Method | Expects | On mismatch |
| --- | --- | --- |
| `db.any` | 0..n rows | never throws |
| `db.many` | 1..n rows | throws on 0 |
| `db.one` | exactly 1 row | throws on 0 or >1 |
| `db.oneOrNone` | 0 or 1 row | throws on >1 |
| `db.none` | 0 rows | throws on any row |
| `db.result` | command result | for row counts on `UPDATE`/`DELETE` |

Choosing the precise method turns a data-integrity violation into an immediate error instead of a silent `undefined` propagating into the domain. `db.any` on a lookup that must return one row is a defect.

### Transactions

Any operation writing more than one row or table runs in a transaction. The use case owns the transactional boundary; the adapter exposes it.

```typescript
await this.db.tx(async (t) => {
  await t.none('INSERT INTO invoices (...) VALUES ($1, ...)', [...]);
  await t.none('INSERT INTO invoice_items (...) SELECT ...', [...]);
});
```

Use `db.task` for read-only multi-query work — it reuses one connection without transaction overhead. Never open a transaction and `await` an HTTP call inside it: the connection is held for the duration of the remote call and the pool starves.

### Connection pool

Configure `max` explicitly and size it against the database's `max_connections` divided by the number of application instances. The default is rarely right for a horizontally-scaled service.

---

## 10) Soft Delete

When rows must be retained for audit, use `deleted_at timestamptz NULL`. Presence of a timestamp means deleted.

```sql
deleted_at timestamptz NULL
```

**Contract:**

- Every read query filters `AND deleted_at IS NULL`. A forgotten filter silently resurrects deleted records — this is the primary hazard of soft delete and the reason it should not be applied to tables that don't need it.
- Unique constraints become partial indexes with `WHERE deleted_at IS NULL` (§5), otherwise a deleted row permanently blocks reuse of its unique value.
- Deletion sets `deleted_at = now()`; it never issues `DELETE`.
- Consider exposing reads through a view (`users_active`) so the filter cannot be omitted.

Do not apply soft delete by default. On tables with no audit or recovery requirement it adds a filter to every query and an index condition to every constraint for no benefit.

---

## 11) Pagination Contract

Offset pagination is the default; it must satisfy the API contract in `api-style-guide.md` §5.

```sql
SELECT u.id, u.email, u.status, u.created_at
FROM users u
WHERE u.tenant_id = $1 AND u.deleted_at IS NULL
ORDER BY u.created_at DESC, u.id DESC
LIMIT $2 OFFSET $3
```

Requirements:

- Deterministic `ORDER BY` with a unique tiebreaker.
- A composite index matching the filter and sort exactly.
- `LIMIT` bounded by the validated DTO (max 100) — never pass an unvalidated client value.

**Offset degrades linearly.** `OFFSET 50000` makes PostgreSQL read and discard 50,000 rows. For large or deep-paginated datasets, switch to keyset pagination:

```sql
WHERE u.tenant_id = $1 AND u.deleted_at IS NULL
  AND (u.created_at, u.id) < ($2, $3)
ORDER BY u.created_at DESC, u.id DESC
LIMIT $4
```

The row-comparison form uses the composite index directly and holds constant cost at any depth.

A separate `COUNT(*)` for `totalRecords` doubles the query cost. On large tables, either cache it, or drop exact totals in favor of keyset pagination with a `hasNext` flag.

---

## 12) Architecture Boundary

The database is an **implementation detail** behind a port.

- The use case depends on a repository interface defined in the Application layer. It never sees SQL, `pg-promise`, or `IDatabase`.
- The adapter (`PostgresUserRepository`) implements that interface and is the only place SQL exists. SQL in a controller, service, or entity is a layering violation.
- The adapter maps rows to domain objects. Raw row shapes must not leak upward — a `snake_case` row object reaching a use case means the mapping was skipped.
- Business rules live in the Domain. Constraints in the schema guarantee **atomic integrity** under concurrency — uniqueness, referential integrity, value ranges — not the full rule set. Both exist; neither replaces the other.

### Reference adapter

```typescript
import { Injectable, Inject } from '@nestjs/common';
import { IDatabase } from 'pg-promise';

@Injectable()
export class PostgresUserRepository implements UserRepository {
  constructor(@Inject('DATABASE_CONNECTION') private readonly db: IDatabase<unknown>) {}

  async findByTenant(tenantId: string, limit: number, offset: number): Promise<User[]> {
    const rows = await this.db.any(
      `SELECT u.id, u.email, u.status, u.created_at
       FROM users u
       WHERE u.tenant_id = $1 AND u.deleted_at IS NULL
       ORDER BY u.created_at DESC, u.id DESC
       LIMIT $2 OFFSET $3`,
      [tenantId, limit, offset],
    );
    return rows.map(UserMapper.toDomain);
  }

  async exists(tenantId: string, email: string): Promise<boolean> {
    const { exists } = await this.db.one(
      `SELECT EXISTS(
         SELECT 1 FROM users
         WHERE tenant_id = $1 AND email = $2 AND deleted_at IS NULL
       ) AS exists`,
      [tenantId, email],
    );
    return exists;
  }
}
```

Note `IDatabase<unknown>` rather than `IDatabase<any>` — `any` disables type checking on every call made through it.

---

## 13) Migrations

- **Forward-only and idempotent-safe.** Each migration is a discrete, reviewed file; never edit one that has been applied to a shared environment.
- **Expand / contract for breaking changes.** Add the new column nullable, backfill, switch the application, then drop the old one in a later release. A rename in one step breaks every running instance of the previous version during deploy.
- **`CREATE INDEX CONCURRENTLY`** on populated tables — a plain `CREATE INDEX` takes an `ACCESS EXCLUSIVE`-blocking share lock that stops writes for the duration.
- **Adding a `NOT NULL` column with a default is safe** on modern PostgreSQL (no table rewrite), but adding a `CHECK` or `FOREIGN KEY` scans the whole table — add it `NOT VALID`, then `VALIDATE CONSTRAINT` separately to avoid a long exclusive lock.
- **Every migration has a tested rollback**, or an explicit statement of why it is forward-fix only (see `create-pull-request` templates).

---

## 14) Review Checklist

- [ ] Tables plural, columns `snake_case`, all English, no quoted mixed-case identifiers
- [ ] Constraints and indexes explicitly named
- [ ] PK is `uuid`, UUIDv7 generated in the application
- [ ] `created_at` / `updated_at` present as `timestamptz NOT NULL`, and `updated_at` is actually maintained
- [ ] Money is `numeric`, never floating point; timestamps are `timestamptz`
- [ ] Enumerations are `text` + `CHECK`, not native `ENUM`
- [ ] Every FK declares its `ON DELETE` action and has an index on the referencing column
- [ ] Uniqueness enforced by a constraint, not by a read-then-write check
- [ ] Partial unique index where soft delete is in use
- [ ] Columns are `NOT NULL` unless null is a meaningful domain state
- [ ] No `SELECT *`; every column listed and qualified
- [ ] Explicit `JOIN ... ON`, no comma joins
- [ ] Paginated queries have a unique tiebreaker and a matching composite index
- [ ] `EXPLAIN ANALYZE` run on new or modified queries against realistic data
- [ ] All values parameterized; dynamic identifiers allowlisted, never interpolated
- [ ] Correct `pg-promise` result method (`one` / `oneOrNone` / `none`), not `any` everywhere
- [ ] Multi-statement writes wrapped in `db.tx`; no remote calls inside a transaction
- [ ] Every read filters `deleted_at IS NULL` where soft delete applies
- [ ] No SQL outside the persistence adapter; rows mapped to domain objects before returning
- [ ] Index creation on populated tables uses `CONCURRENTLY`
