# SQL Style Guide (PostgreSQL + pg-promise)

This guide standardizes **schema**, **migrations**, and **query style** for a Node.js backend using **pg-promise** and **offset pagination**.

---

## 1) Naming & Conventions

### 1.1 Tables and columns

- Use **English**, **plural**, `snake_case`.
    
    - ✅ `users`, `inventory_logs`, `scheduled_events`
        
    - ❌ `User`, `InventoryLog`, `scheduledEvents`
        

### 1.2 Primary keys and foreign keys

- **Primary key column name:** `id`
    
- **Primary key type:** `uuid` (generate as **uuidv7** in the application)
    
- **Foreign key naming:** `<referenced_table_singular>_id`
    
    - Example: `orders.customer_id` references `customers.id`
        

### 1.3 Standard columns (required)

Every table must have:

- `created_at timestamptz NOT NULL DEFAULT now()`
    
- `updated_at timestamptz NOT NULL DEFAULT now()`
    

If you use soft delete:

- `deleted_at timestamptz NULL`
    

---

## 2) Data Types (Postgres)

### 2.1 Strings

- Use `text` by default.
    
- Use `varchar(n)` only if a **hard DB-level limit** is a business requirement.
    

### 2.2 Numbers

- Use `integer` / `bigint` for counts.
    
- Use `numeric(p,s)` for **exact** values (money, measurements).
    
    - Example: `numeric(12,2)` for currency, `numeric(10,4)` for measured quantities.
        
- Avoid `double precision` for currency.
    

### 2.3 Dates & time

- Use `timestamptz` for timestamps.
    
- For time ranges, prefer:
    
    - `>= start AND < end`
        
    - Avoid `BETWEEN` for timestamps.
        

### 2.4 JSON

- Use `jsonb` for schemaless attributes.
    
- Add `GIN` indexes only when you actually query inside the JSON.
    

---

## 3) Constraints & Integrity

### 3.1 NOT NULL

- Use `NOT NULL` whenever the application requires the value.
    

### 3.2 Foreign keys

- Always add FKs unless you have a documented exception.
    
- Choose `ON DELETE` deliberately:
    
    - `RESTRICT` (default safe choice)
        
    - `CASCADE` only for true dependent entities
        
    - `SET NULL` only when the relation is optional
        

### 3.3 UNIQUE

- Use `UNIQUE` for real invariants (e.g., `(tenant_id, email)`).
    

### 3.4 CHECK constraints

- Add `CHECK` for simple, stable rules:
    
    - non-negative values
        
    - bounded statuses
        

`status text NOT NULL CHECK (status IN ('ACTIVE', 'INACTIVE')); quantity numeric(10,4) NOT NULL CHECK (quantity >= 0);`

---

## 4) Indexing Rules

### 4.1 Index based on access patterns

Create indexes that match actual query patterns:

- columns used in `WHERE`
    
- columns used in `ORDER BY` (especially for pagination)
    
- common join keys (FKs)
    

### 4.2 Partial indexes for soft delete

If you filter `deleted_at IS NULL` often:

`CREATE INDEX idx_users_tenant_created_active ON users (tenant_id, created_at DESC, id DESC) WHERE deleted_at IS NULL;`

### 4.3 Composite indexes for offset pagination

If the query is:

- `WHERE tenant_id = ? AND deleted_at IS NULL`
    
- `ORDER BY created_at DESC, id DESC`  
    Then index as:
    

`CREATE INDEX idx_users_tenant_created ON users (tenant_id, created_at DESC, id DESC) WHERE deleted_at IS NULL;`

---

## 5) Migrations (Up/Down)

- Every schema change must be a migration.
    
- Provide a **down** migration when it’s safe.
    
- Avoid destructive changes without a phased plan:
    
    1. add new column
        
    2. backfill
        
    3. switch code
        
    4. drop old column later
        

---

## 6) Query Style & Formatting

### 6.1 Formatting

- Use SQL keywords in **UPPERCASE**.
    
- Break lines after:
    
    - `SELECT`, `FROM`, `JOIN`, `WHERE`, `GROUP BY`, `ORDER BY`
        
- Indent join conditions and boolean logic.
    

### 6.2 Never use `SELECT *`

Always specify selected columns.

### 6.3 Always use explicit joins

Use `JOIN ... ON` (never “join in the WHERE clause”).

### 6.4 Always specify sort direction

If you use `ORDER BY`, always include `ASC` or `DESC`.

### 6.5 Don’t use GROUP BY to “fix duplicates”

If joins duplicate rows, fix the join cardinality or aggregate intentionally.

---

## 7) pg-promise Safety Rules

### 7.1 Always parameterize values

Use `$1, $2, ...` placeholders:

- ✅ `WHERE email = $1`
    
- ❌ string interpolation: `WHERE email = '${email}'`
    

### 7.2 Safe identifiers (table/column names)

If you truly must inject identifiers (rare), use pg-promise formatting:

- `$(table:name)` / `$(column:name)`  
    Never inject raw identifiers.
    

### 7.3 Prefer SQL files for complex queries

Use `pgp.QueryFile` for long queries and to keep TS code clean.

---

## 8) Offset Pagination Contract (Canonical Pattern)

### 8.1 Stable ordering (required)

Offset pagination must always sort by a deterministic, indexed order:

- Recommended: `created_at DESC, id DESC`
    

### 8.2 Query

`SELECT   u.id,   u.tenant_id,   u.name,   u.email,   u.status,   u.created_at,   u.updated_at FROM users u WHERE u.tenant_id = $1   AND u.deleted_at IS NULL ORDER BY u.created_at DESC, u.id DESC LIMIT $2 OFFSET $3;`

### 8.3 Matching index

`CREATE INDEX idx_users_tenant_created_active ON users (tenant_id, created_at DESC, id DESC) WHERE deleted_at IS NULL;`

---

## 9) Canonical Query Patterns

### 9.1 “Exists” instead of `COUNT(*) > 0`

`SELECT EXISTS (   SELECT 1   FROM users u   WHERE u.tenant_id = $1     AND u.email = $2     AND u.deleted_at IS NULL ) AS exists;`

### 9.2 Insert + RETURNING

`INSERT INTO users (   id,   tenant_id,   name,   email,   status,   created_at,   updated_at ) VALUES (   $1, $2, $3, $4, $5, now(), now() ) RETURNING   id, tenant_id, name, email, status, created_at, updated_at;`

### 9.3 Update + RETURNING

`UPDATE users SET name = $3,     updated_at = now() WHERE id = $1   AND tenant_id = $2   AND deleted_at IS NULL RETURNING   id, tenant_id, name, email, status, created_at, updated_at;`

### 9.4 Soft delete

`UPDATE users SET deleted_at = now(),     updated_at = now() WHERE id = $1   AND tenant_id = $2   AND deleted_at IS NULL;`

### 9.5 Aggregation (intentional)

`SELECT   o.customer_id,   COUNT(*) AS orders_count,   SUM(o.total_amount) AS total_spent FROM orders o WHERE o.tenant_id = $1   AND o.created_at >= $2   AND o.created_at <  $3 GROUP BY o.customer_id ORDER BY total_spent DESC;`

---

## 10) DDL Template (Recommended)

`CREATE TABLE users (   id uuid PRIMARY KEY,   tenant_id uuid NOT NULL,   name text NOT NULL,   email text NOT NULL,   status text NOT NULL CHECK (status IN ('ACTIVE', 'INACTIVE')),   created_at timestamptz NOT NULL DEFAULT now(),   updated_at timestamptz NOT NULL DEFAULT now(),   deleted_at timestamptz NULL,    CONSTRAINT users_tenant_email_uk UNIQUE (tenant_id, email),   CONSTRAINT users_tenant_fk FOREIGN KEY (tenant_id)     REFERENCES tenants (id)     ON DELETE RESTRICT );  CREATE INDEX idx_users_tenant_created_active ON users (tenant_id, created_at DESC, id DESC) WHERE deleted_at IS NULL;`

---

## 11) UUIDv7 Rule

- Primary keys are `uuid`.
    
- Generate **uuidv7 in the application** and pass it as the `id` value.
    
- Keep the DB default for `id` unset unless you have a standardized DB-side uuidv7 function.
    

---

## 12) pg-promise Examples (TypeScript)

### 12.1 List with offset pagination

``type ListUsersParams = {   tenantId: string;   limit: number;   offset: number; };  export async function listUsers(db: any, p: ListUsersParams) {   return db.any(     `     SELECT       u.id,       u.tenant_id,       u.name,       u.email,       u.status,       u.created_at,       u.updated_at     FROM users u     WHERE u.tenant_id = $1       AND u.deleted_at IS NULL     ORDER BY u.created_at DESC, u.id DESC     LIMIT $2     OFFSET $3     `,     [p.tenantId, p.limit, p.offset],   ); }``

### 12.2 Count + list (for metadata)

``export async function listUsersWithCount(db: any, p: ListUsersParams) {   const sqlCount = `     SELECT COUNT(*)::int AS total     FROM users u     WHERE u.tenant_id = $1       AND u.deleted_at IS NULL   `;    const sqlList = `     SELECT       u.id,       u.tenant_id,       u.name,       u.email,       u.status,       u.created_at,       u.updated_at     FROM users u     WHERE u.tenant_id = $1       AND u.deleted_at IS NULL     ORDER BY u.created_at DESC, u.id DESC     LIMIT $2     OFFSET $3   `;    return db.tx(async (t: any) => {     const { total } = await t.one(sqlCount, [p.tenantId]);     const rows = await t.any(sqlList, [p.tenantId, p.limit, p.offset]);     return { total, rows };   }); }``
