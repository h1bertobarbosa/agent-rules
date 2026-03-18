---
name: api-style-guide
description: Specialist in RESTful API design following rigorous resource modeling standards, command-based versioning, stable pagination, and integration with NestJS under the principles of Clean Architecture.
---

# API Style Guide Specialist

## 1. What I do

- I guide resource modeling using **plural names in English** and **kebab-case**.

- I define the endpoint structure following a clear distinction between **CRUD (PATCH)** and **Commands (POST)**.

- I establish response standards for success, error (standardized format), and pagination (with metadata and links).

- I ensure the correct application of **HTTP Status Codes** according to the semantics of the operation.

- I configure DTO validations in NestJS and resilience strategies for external calls.

## 2. When to use me

- When creating new modules or microservices that expose HTTP interfaces.

- During Code Review to ensure that the API does not have weak models or poorly formatted endpoints.

- To define integration contracts between frontend and backend or between services.

3. Rules and Standards (Mandatory)

- **Correct:** `/customer-invoices` | **Incorrect:** `/customerInvoices` or `/customer_invoices`.

- **Hierarchy:** Maximum of **2 nesting levels** (e.g., `/customers/:id/invoices`). For deeper levels, use Query Params or higher-level resources.

- **Verbs:** Use of `PUT` is prohibited. Partial updates must use `PATCH`. Business actions use `POST /resource/:id/action-name`.

- **Tenancy:** If the resource exists but does not belong to the user/tenant, return **404 Not Found** (to prevent information leaks) instead of 403.

- **Pagination:** Returning `data`, `meta`, and `links` is mandatory. The `page` and `limit` parameters must be validated (max limit 100).

- **Persistence:** Every paginated query must have deterministic sorting and a corresponding index in the database.

## 4. Reference Examples

### Controller Definition (NestJS + Clean Arch)

```typescript
@Controller('scheduled-events')
export class ScheduledEventsController {
  constructor(private readonly cancelEventUseCase: CancelEventUseCase) {}

  @Post(':id/cancel')
  @HttpCode(HttpStatus.OK)
  async cancel(@Param('id') id: string, @Body() dto: CancelEventDto) {
    // O UseCase lida com a lógica, o Controller apenas com o protocolo HTTP
    await this.cancelEventUseCase.execute({ id, ...dto });
    return { message: 'Event successfully cancelled' };
  }

  @Get()
  async list(@Query() query: PaginationQueryDto) {
    // Retorno seguindo o padrão de paginação exigido
    return {
      data: [],
      meta: { totalRecords: 0, currentPage: query.page, totalPages: 0, limit: query.limit },
      links: { first: '...', prev: null, next: null, last: '...' }
    };
  }
}
```

### Validação de DTO
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
