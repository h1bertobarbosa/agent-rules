---
name: postgres-guideline
description: Standardization guide for PostgreSQL and pg-promise following Clean Architecture and DDD principles. Focused on performance, integrity, and schema consistency.
---

# PostgreSQL & pg-promise Style Guide

## 1. What I do

- Standardize the creation of schemas, tables, and columns (Naming Conventions).

- Define recommended data types for different domains (Financial, Audit, Schemaless).

- Establish integrity rules (FKs, Constraints, Unique) and indexing patterns for performance.

- Guide the writing of secure and performant SQL queries with `pg-promise`.

- Implement the canonical contract for pagination via Offset and logical deletion (Soft Delete).

## 2. When to use me

- When creating new database migrations.

- When developing repositories (Persistence Adapters) in NestJS.

- During data model design to ensure that the infrastructure supports DDD domain rules.

- In code reviews focused on optimizing SQL queries.

## 3. Mandatory Rules and Standards

- **Naming:** Tables in **plural**, columns in **snake_case**, everything in **English**.

- **IDs:** Use `uuid` (UUIDv7 generated in the application/Use Case) as the Primary Key.

- **Auditing:** Every table must have `created_at` and `updated_at` with `timestamptz`.

- **Performance:** - The use of `SELECT *` is prohibited.

- Always use **Explicit Joins** (`JOIN ... ON`).

- Pagination requires a deterministic `ORDER BY` (e.g., `created_at DESC, id DESC`).

- **Security:** Never use string interpolation in queries. Use the `$1, $2` placeholders from `pg-promise`.

- **Architecture:** The database should be treated as an implementation detail (Adapter Interface). Complex business rules reside in the Domain, while the database ensures atomic integrity via Constraints.

## 4. Reference Examples
### 4.1 Definição de Tabela (Migration)
```sql
CREATE TABLE users (
    id uuid PRIMARY KEY,
    tenant_id uuid NOT NULL,
    email text NOT NULL,
    status text NOT NULL CHECK (status IN ('ACTIVE', 'INACTIVE')),
    created_at timestamptz NOT NULL DEFAULT now(),
    updated_at timestamptz NOT NULL DEFAULT now(),
    deleted_at timestamptz NULL,
    CONSTRAINT users_tenant_email_uk UNIQUE (tenant_id, email),
    CONSTRAINT users_tenant_fk FOREIGN KEY (tenant_id) REFERENCES tenants (id) ON DELETE RESTRICT
);

-- Índice composto para suporte à paginação e filtro por tenant
CREATE INDEX idx_users_list_active ON users (tenant_id, created_at DESC, id DESC) 
WHERE deleted_at IS NULL;
```

### 4.2 Repositório NestJS (Persistence Adapter)
```sql
import { Injectable, Inject } from '@nestjs/common';
import { IDatabase } from 'pg-promise';

@Injectable()
export class PostgresUserRepository {
  // Injeção de dependência por Token do pg-promise
  constructor(@Inject('DATABASE_CONNECTION') private readonly db: IDatabase<any>) {}

  async findByTenant(tenantId: string, limit: number, offset: number) {
    const query = `
      SELECT u.id, u.email, u.status, u.created_at
      FROM users u
      WHERE u.tenant_id = $1 AND u.deleted_at IS NULL
      ORDER BY u.created_at DESC, u.id DESC
      LIMIT $2 OFFSET $3
    `;
    return this.db.any(query, [tenantId, limit, offset]);
  }

  async exists(tenantId: string, email: string): Promise<boolean> {
    const result = await this.db.one(
      'SELECT EXISTS(SELECT 1 FROM users WHERE tenant_id = $1 AND email = $2) as exists',
      [tenantId, email]
    );
    return result.exists;
  }
}
```
