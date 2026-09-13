# RESTful API Style Guide — Technical Standardization (Practical, "Right vs Wrong")

This guide is written to be enforceable in code reviews. It covers resource modeling, endpoint structure, response contracts, and integration concerns for HTTP APIs built with NestJS under Clean Architecture.

Applies when: creating a module or microservice that exposes an HTTP interface, reviewing an API surface, or defining an integration contract between frontend and backend or between services.

See also: `general-guidelines.md` for naming conventions.

---

## 1) Resource Modeling

Resources are **nouns**, always **plural**, always **English**, always **kebab-case**.

✅ **Right**

```
/customer-invoices
/scheduled-events
/shipment-orders
```

❌ **Wrong**

```
/customerInvoices        # camelCase
/customer_invoices       # snake_case
/customer-invoice        # singular
/notas-fiscais           # not English
/getCustomerInvoices     # verb in the path
```

### Nesting

**Maximum of 2 levels.** Deeper hierarchies become unnavigable and couple the URL to the aggregate structure.

✅ **Right**

```
/customers/:id/invoices
/invoices?customerId=:id&status=overdue     # filtering instead of nesting
```

❌ **Wrong**

```
/customers/:id/invoices/:invoiceId/items/:itemId/taxes
```

When you need a third level, either promote the deep resource to a top-level one (`/invoice-items/:id`) or express the relationship with query params.

### Identifiers

Path identifiers are opaque strings. Never expose sequential database primary keys in public APIs when the resource is tenant-scoped — they leak volume and enable enumeration. Prefer UUID or ULID.

---

## 2) HTTP Methods

| Operation | Method | Notes |
| --- | --- | --- |
| List | `GET /resources` | Always paginated (§5) |
| Read one | `GET /resources/:id` | |
| Create | `POST /resources` | Returns 201 + `Location` |
| Partial update | `PATCH /resources/:id` | The **only** update verb allowed |
| Delete | `DELETE /resources/:id` | |
| Business action | `POST /resources/:id/action-name` | See Commands below |

### `PUT` is prohibited

`PUT` requires full-representation replacement semantics, which almost no client implements correctly — it silently nulls fields the caller omitted. Every update is a `PATCH`.

### CRUD vs Commands

A state transition driven by a business rule is **not** a field update. Model it as a command: `POST` on a sub-path named after the action in kebab-case.

✅ **Right**

```
POST /scheduled-events/:id/cancel
POST /invoices/:id/settle
POST /shipment-orders/:id/dispatch
```

❌ **Wrong**

```
PATCH /scheduled-events/:id   { "status": "CANCELLED" }
```

The `PATCH` version lets the client drive the state machine and bypasses the invariants that decide whether cancellation is legal at all. The command version routes to a use case that enforces them.

**Rule:** if a field change has preconditions, side effects, or emits a domain event, it is a command.

---

## 3) HTTP Status Codes

| Code | When |
| --- | --- |
| `200 OK` | Successful `GET`, `PATCH`, or command with a response body |
| `201 Created` | `POST` that created a resource — include `Location` header |
| `202 Accepted` | Command dispatched for asynchronous processing |
| `204 No Content` | `DELETE`, or a command with nothing to return |
| `400 Bad Request` | Malformed payload, failed DTO validation |
| `401 Unauthorized` | Missing or invalid credentials |
| `403 Forbidden` | Authenticated, resource visible to the tenant, action not permitted |
| `404 Not Found` | Resource does not exist **or does not belong to the caller's tenant** |
| `409 Conflict` | Violated invariant or state-machine transition (e.g. cancelling a settled invoice) |
| `422 Unprocessable Entity` | Syntactically valid payload that fails a business rule |
| `429 Too Many Requests` | Rate limit — include `Retry-After` |

### Tenancy: 404, never 403

If a resource exists but belongs to another tenant, return **404**. A 403 confirms the ID exists, which leaks record counts and enables enumeration of other tenants' data.

✅ **Right** — repository scopes by tenant; a miss is indistinguishable from a nonexistent ID.

```typescript
const invoice = await this.invoiceRepository.findByIdAndTenant(id, tenantId);
if (!invoice) throw new NotFoundException();
```

❌ **Wrong**

```typescript
const invoice = await this.invoiceRepository.findById(id);
if (!invoice) throw new NotFoundException();
if (invoice.tenantId !== tenantId) throw new ForbiddenException(); // leaks existence
```

Reserve 403 for the case where the caller can legitimately see the resource but lacks permission for this specific action.

---

## 4) Error Contract

Every error response uses the same shape. Clients must never have to parse prose.

```json
{
  "error": {
    "code": "INVOICE_ALREADY_SETTLED",
    "message": "Invoice cannot be cancelled after settlement.",
    "details": [
      { "field": "status", "issue": "expected one of: PENDING, OVERDUE" }
    ],
    "traceId": "01HQ8X2K9M4N7P"
  }
}
```

- `code` — stable SCREAMING_SNAKE_CASE identifier. This is the contract; clients branch on it. Never change one without versioning.
- `message` — human-readable, for logs and developer consoles. Not for end-user display.
- `details` — optional, populated for validation failures.
- `traceId` — always present, correlates to logs.

❌ **Wrong**

```json
{ "message": "erro ao cancelar" }
{ "success": false, "error": "Something went wrong" }
{ "statusCode": 500, "message": ["status must be a valid enum value"] }
```

**Never leak internals.** Stack traces, SQL fragments, ORM error text, and upstream service payloads must not cross the HTTP boundary. Map them to a generic `500` with a `traceId` and log the detail server-side.

Domain errors are translated to HTTP in the adapter layer — never throw `HttpException` from a use case or entity (see `clean-arch.md`).

---

## 5) Pagination

Every collection endpoint is paginated. No exceptions — an endpoint that returns "all" today returns a timeout in production.

### Response shape (mandatory)

```json
{
  "data": [],
  "meta": {
    "totalRecords": 0,
    "currentPage": 1,
    "totalPages": 0,
    "limit": 20
  },
  "links": {
    "first": "/invoices?page=1&limit=20",
    "prev": null,
    "next": "/invoices?page=2&limit=20",
    "last": "/invoices?page=5&limit=20"
  }
}
```

Returning a bare array from a list endpoint is a defect — it leaves no room to add metadata without a breaking change.

### Parameter validation

`page` and `limit` are always validated. **`limit` is capped at 100.**

```typescript
export class PaginationQueryDto {
  @Transform(({ value }) => parseInt(value))
  @IsInt()
  @Min(1)
  page: number = 1;

  @Transform(({ value }) => parseInt(value))
  @IsInt()
  @Min(1)
  @Max(100)
  limit: number = 20;
}
```

### Stability

**Every paginated query needs deterministic sorting and a matching index.**

Without a total order, rows shift between pages and records are silently skipped or duplicated. Sorting by a non-unique column is not deterministic — always append a unique tiebreaker.

✅ **Right**

```sql
ORDER BY created_at DESC, id DESC     -- index: (created_at DESC, id DESC)
```

❌ **Wrong**

```sql
ORDER BY created_at DESC              -- ties reorder between queries
-- no ORDER BY at all
```

For large or frequently-changing datasets, prefer cursor pagination (`?cursor=<opaque>&limit=20`) over offset — `OFFSET 50000` scans and discards 50,000 rows on every request. Keep the same `data`/`meta`/`links` envelope; `meta` carries `nextCursor` instead of page counts.

---

## 6) Versioning

Version the **contract**, not every endpoint. Prefix the path: `/v1/invoices`.

### What is a breaking change

Breaking (requires a new version):

- Removing or renaming a field, endpoint, or error `code`
- Changing a field's type or its nullability from nullable to required
- Adding a required request field
- Narrowing an enum's accepted values
- Changing the meaning of an existing value

Non-breaking (ship in place):

- Adding an optional request field
- Adding a response field
- Adding a new endpoint or a new error `code` for a previously generic failure
- Widening an enum's returned values — **only if** clients are documented to tolerate unknown values

### Commands as a versioning escape hatch

When a business action's semantics change, introduce a new command rather than versioning the whole API: `POST /invoices/:id/settle-partial` alongside the existing `settle`. This keeps the change additive and lets the old command be deprecated on its own schedule.

Deprecation: mark with a `Deprecation` and `Sunset` header, keep serving for at least one release cycle, and never remove without a consumer audit.

---

## 7) Request Validation

Validation happens at the DTO boundary, in the adapter layer. It is a **protocol** concern — shape, type, and format. It does not replace domain invariants.

```typescript
export class CancelEventDto {
  @IsString()
  @IsNotEmpty()
  @MaxLength(500)
  reason: string;

  @IsOptional()
  @IsISO8601()
  effectiveAt?: string;
}
```

Global pipe configuration is mandatory:

```typescript
app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,           // strip unknown properties
    forbidNonWhitelisted: true, // 400 on unknown properties
    transform: true,
  }),
);
```

`whitelist` without `forbidNonWhitelisted` silently drops a client's typo'd field and returns 200 — the caller thinks the value was applied.

**DTO validation is not business validation.** `@IsInt() @Min(1) quantity` proves the payload is well-formed; only the entity can decide whether that quantity is available in stock. Both layers validate; neither substitutes for the other.

---

## 8) Controllers

Controllers handle HTTP protocol only: parse, delegate, serialize. No business logic, no repository access, no orchestration.

✅ **Right**

```typescript
@Controller('scheduled-events')
export class ScheduledEventsController {
  constructor(private readonly cancelEventUseCase: CancelEventUseCase) {}

  @Post(':id/cancel')
  @HttpCode(HttpStatus.OK)
  async cancel(@Param('id') id: string, @Body() dto: CancelEventDto) {
    await this.cancelEventUseCase.execute({ id, ...dto });
    return { message: 'Event successfully cancelled' };
  }

  @Get()
  async list(@Query() query: PaginationQueryDto) {
    return this.listEventsUseCase.execute(query);
  }
}
```

❌ **Wrong**

```typescript
@Patch(':id')
async cancel(@Param('id') id: string, @Body() body: any) {
  const event = await this.repo.findById(id);        // repository in controller
  if (event.status === 'DONE') throw new Error();    // business rule in controller
  event.status = 'CANCELLED';                        // state machine in controller
  return this.repo.save(event);
}
```

---

## 9) Outbound Calls and Resilience

Every call to an external service is a failure mode. Treat it as one.

**Mandatory for each outbound integration:**

- **Timeout.** Explicit, per-call, never the library default. An un-timeouted call holds a request thread until the socket dies.
- **Retry with exponential backoff and jitter** — and only for idempotent operations or calls carrying an idempotency key. Retrying a non-idempotent `POST` duplicates the side effect.
- **Circuit breaker.** After N consecutive failures, fail fast instead of queueing requests against a dead dependency.
- **Bounded retries.** Cap attempts; an unbounded retry loop turns a partial outage into a self-inflicted DDoS on the recovering service.

```typescript
const response = await firstValueFrom(
  this.http.post(url, payload, {
    timeout: 3000,
    headers: { 'Idempotency-Key': commandId },
  }).pipe(
    retry({ count: 3, delay: (_, n) => timer(2 ** n * 100 + Math.random() * 100) }),
  ),
);
```

**Failure semantics:** an upstream timeout is `504 Gateway Timeout` or `503 Service Unavailable` — never a `500` and never a silent empty result. A degraded dependency must be visible to the caller, not papered over with a default value.

External clients are **ports** implemented in the infrastructure layer. The use case depends on the interface and knows nothing about HTTP, retries, or the breaker.

---

## 10) Review Checklist

- [ ] Resources are plural, English, kebab-case
- [ ] Nesting is at most 2 levels
- [ ] No `PUT`; partial updates use `PATCH`
- [ ] Business actions are `POST /resource/:id/action-name`, not status `PATCH`es
- [ ] Cross-tenant access returns 404, not 403
- [ ] Errors use the standard envelope with a stable `code` and a `traceId`
- [ ] No stack traces, SQL, or upstream payloads in responses
- [ ] Every list endpoint returns `data` / `meta` / `links`
- [ ] `page` and `limit` validated; `limit` capped at 100
- [ ] Paginated queries have deterministic sort + matching index
- [ ] `ValidationPipe` uses `whitelist` **and** `forbidNonWhitelisted`
- [ ] Controllers contain no business logic or repository access
- [ ] Every outbound call has a timeout; retries only where idempotent
