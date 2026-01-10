# Coding Standards — Clean Architecture (TypeScript)

## 1. Language & Naming

**All source code must be written in English.**

### Naming conventions

| Element                       | Convention           | Example                                  |
| ----------------------------- | -------------------- | ---------------------------------------- |
| Classes & Interfaces          | PascalCase           | `OrderService`, `UserRepository`         |
| Methods, functions, variables | camelCase            | `calculateTotal`, `userId`               |
| Files & folders               | kebab-case           | `order-service.ts`, `inventory-adapter/` |
| Constants                     | SCREAMING_SNAKE_CASE | `MAX_RETRY_COUNT`                        |

### Abbreviations

Avoid abbreviations unless they are:

- Industry standard
    
- Unambiguous
    
- Widely known
    

**Allowed examples**  
`id`, `url`, `http`, `api`, `dto`, `sql`, `db`, `ui`, `sku`, `ttl`, `jwt`

**Disallowed**  
`tmp`, `calc`, `info`, `obj`, `data1`, `misc`

### Name length

- Prefer **10–30 characters**
    
- If a name exceeds ~30 chars, the abstraction is probably wrong
    

---

## 2. Functions & Methods

### Single responsibility

Every function must perform **one clear action**, reflected in its name.

- Must start with a **verb**
    
- Must not mix multiple responsibilities
    

Good:

`calculateTotal() validateEmail() persistOrder()`

Bad:

`order() data() process()`

---

### Parameters

- Prefer **≤ 3 positional parameters**
    
- If more are needed → use an **options object**
    

Good:

`createUser({ name, email, role })`

Bad:

`createUser(name, email, role, status, createdAt)`

---

### Commands vs Queries

Follow **CQRS at function level**:

|Type|Rule|
|---|---|
|Query|Must NOT modify domain state|
|Command|May mutate state|

Logging, metrics, and tracing are allowed in both.

A function may **not** both mutate state and return domain data.

---

### Boolean flags

Avoid boolean flags that change behavior.

Bad:

`calculateTotal(order, true)`

Good:

`calculateWithTax(order) calculateWithoutTax(order)`

If variation is legitimate:

`calculateTotal(order, { includeTax: true })`

---

## 3. Control Flow

### Conditionals

- Use **guard clauses**
    
- Avoid deep nesting (>2 levels)
    

Good:

`if (!user) return error if (!user.isActive) return error`

Bad:

`if (user) {   if (user.isActive) {     if (user.hasPermission) {`

Prefer:

- `switch`
    
- maps
    
- polymorphism
    

---

## 4. Size Limits (Soft Rules)

|Element|Preferred limit|
|---|---|
|Function|≤ 40 lines|
|Class|≤ 350 lines|

Exceeding this requires:

- clear justification
    
- or refactor by responsibility
    

These are **quality thresholds**, not mechanical limits.

---

## 5. Variables

- Never declare multiple variables on one line
    
- Declare variables as close as possible to their use
    
- Always prefer `const`
    
- Avoid mutation unless required
    

---

## 6. Comments

Avoid comments that repeat what the code says.

Comments are **mandatory** when:

- Business rules are non-obvious
    
- There is a workaround
    
- Performance or timezones are involved
    
- The code is intentionally surprising
    

---

## 7. Magic Numbers & Constants

All meaningful literals must be named.

Bad:

`if (attempts > 3)`

Good:

`if (attempts > MAX_LOGIN_ATTEMPTS)`

---

## 8. Clean Architecture Boundaries

### Dependency rule

Inner layers must never depend on outer layers.

**External resources include**

- Database
    
- HTTP
    
- Message brokers
    
- Filesystem
    
- Time
    
- Randomness
    
- Environment variables
    

These must be accessed through **ports (interfaces)**.

`UseCase → Port → Adapter → Infrastructure`

Use cases must not import:

- ORM
    
- HTTP clients
    
- Redis
    
- filesystem
    
- environment variables
    
Single Responsibility and Layering (CleanArch)

**Golden Rule:** _all dependencies point inward_.

**Domain**

- Entities, VOs, invariant rules.

- **No** NestJS, **no** TypeORM/Prisma, **no** external libraries.

- Domain-specific errors (e.g., `InsufficientStockError`).

**Application**

- Use cases and ports (interfaces).

- Flow orchestration (transactions, gateway calls, event publishing).

- Returns well-defined **results** (e.g., `Either`/`Result`) and not `HttpException`.

**Infrastructure**

- Implementations: PostgreSQL repositories, caching, queues, integrations.

- Serialization details, query builder, SQL, retries.

**Interface/Adapters (NestJS)**

- Controllers, DTOs, pipes, guards, interceptors.

- Request Maps → UseCase Input and Output → Response.

- No business rules here.

**Review Checklist**

- Controller has `@Body/@Query` + validation + calls use case + returns DTO ✅

- Use case contains business rules/flow ✅

- Repository does not "decide" business rules, only persists/queries ✅

- Domain does not import any infrastructure ✅

---

### 1.3 Coupling and Contracts: Ports and Adapters

**Rules**

- Use case depends on interfaces: `OrderRepository`, `CachePort`, `EventBusPort`

- Infrastructure implementation: `PgOrderRepository`, `RedisCacheAdapter`, `OtelEventBusAdapter`

**Best Practices**

- **Do not** insert `DataSource`, `PrismaClient`, `RedisClient` directly into the UseCase.

- Prefer **small interfaces** (ISPs). For example, instead of a giant `CacheService`, use `CacheGetPort` or `CacheSetPort`.
---

## 9. Composition

Prefer:

- composition
    
- dependency injection
    
- interfaces
    

Avoid:

- deep inheritance
    
- framework-coupled domain objects
    

---

# 🔍 Review Checklist (Every Task)

Before merging:

### Build & Tests

- `npm test`
    
- `npm run build`
    
- `npm run lint`
    

### Quality gates

- Test coverage:
    
    - Domain & Use Cases ≥ 90%
        
    - Overall ≥ 80%
        
- No failing lint rules
    
- No unused imports
    
- No unused variables
    
- No TODOs without ticket reference
    

### Architecture

- No use case imports adapters, ORM, or frameworks
    
- All external resources are behind ports
    

### Code health

- No hardcoded business values
    
- No duplicated logic
    
- No boolean behavior flags
    
- No unclear names
    

---

# 📜 Logging Rules

- Never write logs to files
    
- Never log:
    
    - Names
        
    - Emails
        
    - Addresses
        
    - Payment data
        
    - Tokens
        
- Logs must be short, factual, and structured
    

### Errors

Errors must be:

- handled **or**
    
- propagated to a centralized handler
    

Never:

- swallow errors
    
- log the same error multiple times