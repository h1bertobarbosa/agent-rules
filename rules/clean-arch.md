# DDD + Clean Architecture — Technical Standardization Guide (Practical, “Right vs Wrong”)

This guide is written to be enforceable in code reviews. Every rule below is meant to reduce coupling, keep the Domain independent, and make change cheaper.

---

## 1) Architecture and Dependency Flow

### Layers and responsibilities

#### **Domain (Business Rules)**

**Owns:** invariants, entities, value objects, aggregates, domain events, domain services (pure business), domain errors.  
**Must NOT know:** HTTP, controllers, ORMs, database schemas, queues, frameworks, DI containers.

✅ **Right**

- “Can we create an Order with invalid state?” → **Domain answers**
    
- “How do we compute shipping eligibility?” → **Domain answers**
    

❌ **Wrong**

- Entity imports `typeorm`, `mongoose`, `axios`, `@nestjs/*`
    
- Domain throws `HttpException`
    

---

#### **Application (Use Cases / Orchestration)**

**Owns:** use cases, transactions, orchestration across repositories/services, authorization at use-case level, mapping between DTOs and domain objects, port interfaces (repositories/gateways).  
**Must NOT own:** persistence details, HTTP concerns, ORM models.

✅ **Right**

- `CreateOrderUseCase` loads dependencies via ports, enforces flow, saves aggregate.
    

❌ **Wrong**

- Use case queries DB using ORM directly, returns ORM entities to controller.
    

---

#### **Infrastructure (Implementation / I/O)**

**Owns:** repository implementations, ORM mappings, database adapters, external API clients, message bus adapters, cache providers.  
**Depends on:** Application ports + Domain models.

✅ **Right**

- `TypeOrmOrderRepository implements OrderRepository`
    

❌ **Wrong**

- Infrastructure defines business rules (“max discount”) or changes entity state “just to fit DB”.
    

---

#### **API / Interface (Controllers, Presenters, CLI, GraphQL)**

**Owns:** input/output DTOs, request parsing, auth middleware, validation _shape_, formatting, status codes.  
**Depends on:** Application use cases.

✅ **Right**

- Controller validates request shape, builds command DTO, calls use case, formats response.
    

❌ **Wrong**

- Controller contains business decisions (“if VIP then discount 30%”).
    

---

### Golden Rule: Dependencies point inward

**Dependency Rule:** `API → Application → Domain` and `Infrastructure → Application → Domain`  
Domain is the center: it should compile without frameworks or infrastructure.

#### Why this matters

- Your business rules survive framework swaps.
    
- Tests become cheap and fast.
    
- “Database is a detail”, not the model.
    

---

### Dependency Inversion with Ports (Interfaces)

**Ports live in Application (or Domain if truly pure). Implementations live in Infrastructure.**

✅ **Right (port in Application)**

`// application/ports/order-repository.ts import { Order } from "../../domain/order/order";  export interface OrderRepository {   save(order: Order): Promise<void>;   findById(id: string): Promise<Order | null>;   existsByExternalRef(ref: string): Promise<boolean>; }`

❌ **Wrong (port in Infrastructure)**

`// infrastructure/db/order-repository.ts export interface OrderRepository { /* ... */ } // now app depends on infra`

---

## 2) DDD Design Tactics

### 2.1 Entities — rich, valid, invariant-protected (avoid anemic models)

**Rule E1 — Entity must protect its invariants**

- Construct via factory/static constructor or validated constructor.
    
- State changes only through intention-revealing methods.
    
- No public setters.
    

✅ **Right**

`// domain/order/order.ts import { Money } from "../shared/money"; import { OrderItem } from "./order-item";  export class Order {   private constructor(     public readonly id: string,     private status: "DRAFT" | "CONFIRMED",     private items: OrderItem[],     private total: Money,   ) {}    static create(id: string): Order {     return new Order(id, "DRAFT", [], Money.zero("BRL"));   }    addItem(item: OrderItem) {     if (this.status !== "DRAFT") throw new Error("ORDER_NOT_EDITABLE");     this.items.push(item);     this.recalculateTotal();   }    confirm() {     if (this.items.length === 0) throw new Error("EMPTY_ORDER");     this.status = "CONFIRMED";   }    private recalculateTotal() {     this.total = this.items.reduce(       (acc, item) => acc.add(item.subtotal()),       Money.zero(this.total.currency),     );   } }`

❌ **Wrong (anemic)**

`export class Order {   id!: string;   status!: string;   items!: any[];   total!: number; } // logic scattered in services/controllers`

**Rule E2 — Entities don’t “validate input DTOs”**  
Entities validate _business meaning_, not HTTP payload shape.

---

### 2.2 Value Objects (VO) — immutable, encapsulate logic, equality by value

**Rule V1 — VO is immutable**

- Freeze state; no setters.
    
- Validate on creation; store normalized format.
    

**Rule V2 — Put logic inside VO**

- Formatting, parsing, comparisons, derived values.
    

✅ **Right (CPF VO example)**

``// domain/shared/cpf.ts export class CPF {   private constructor(private readonly value: string) {}    static create(raw: string): CPF {     const normalized = raw.replace(/\D/g, "");     if (normalized.length !== 11) throw new Error("CPF_INVALID");     // (optional) add CPF check-digit algorithm here     return new CPF(normalized);   }    format(): string {     const v = this.value;     return `${v.slice(0,3)}.${v.slice(3,6)}.${v.slice(6,9)}-${v.slice(9)}`;   }    equals(other: CPF): boolean {     return this.value === other.value;   }    toString(): string {     return this.value;   } }``

❌ **Wrong**

`type CPF = string; // logic duplicated everywhere, no invariant`

✅ **Right (Address VO)**

- `Address` validates completeness and normalizes (zip, state codes), and exposes `equals()`.
    

---

### 2.3 DTOs — data in/out only, never leak Entities

**Rule D1 — API contracts use DTOs, not Entities**

- DTOs are _shape_, not behavior.
    
- They can be validated syntactically (schema/class-validator/zod).
    
- They are not passed into Domain unchanged.
    

✅ **Right**

- `CreateOrderRequestDTO` (API) → `CreateOrderCommand` (Application) → Domain objects
    

❌ **Wrong**

- Controller returns `Order` entity directly (leaks invariants and internal fields)
    

---

### 2.4 Validations — Syntactic vs Semantic

**Syntactic (Input) validation**

- “Is it present? type? length? enum? format?”
    
- Lives in API layer (or boundary of Application for non-HTTP entrypoints).
    

**Semantic (Business) validation**

- “Is it allowed? does it violate rules? is it consistent with domain state?”
    
- Lives in Domain (entities/VO/services) or Application (cross-aggregate checks via repositories).
    

✅ **Right**

- API checks: `email is valid`, `items is non-empty array`
    
- Domain checks: `cannot confirm empty order`, `cannot add item after confirmed`
    
- Application checks: `externalRef must be unique` (requires repo)
    

❌ **Wrong**

- Domain uses `class-validator` decorators
    
- Controller enforces “max discount” business rule
    

---

## 3) Mandatory Design Patterns

### 3.1 Repository Pattern — only for Aggregates

**Rule R1 — Only aggregate roots have repositories**

- You persist/load aggregates as a consistency boundary.
    

✅ **Right**

- `OrderRepository` persists `Order` aggregate root.
    

❌ **Wrong**

- `OrderItemRepository` (usually wrong) + separate writes that break invariants.
    

**Rule R2 — Repository returns Domain objects, not ORM models**

- ORM models are infra-only.
    

---

### 3.2 Factory / Builder — for complex creation

Use when:

- Construction has many invariants or optional branches
    
- You must enforce default values and normalization
    
- There’s more than “just new()”
    

✅ **Right**

- `OrderItem.create(productId, qty, price)` ensures qty > 0, price currency consistent.
    

❌ **Wrong**

- Massive constructor with 12 params used incorrectly everywhere.
    

**Rule F1 — Prefer named constructors over raw constructors**

- `Order.create()`, `Money.from()`, `Address.create()`
    

---

### 3.3 Domain Services — logic that doesn’t belong to a single entity

Use when:

- Logic spans multiple entities/VOs but is still pure domain
    
- It doesn’t naturally fit one aggregate root without making it “god object”
    

✅ **Right**

- `PricingService.calculate(orderItems, customerTier) -> Money` (pure)
    

❌ **Wrong**

- `OrderDomainService` that does DB calls (that’s Application/Infrastructure)
    

**Rule S1 — Domain services must be side-effect free**  
If it needs repositories, it’s no longer pure domain—move orchestration to Application.

---

## 4) Testing Strategy (Pyramid)

### 4.1 Unit Tests — Domain first (where logic lives)

**Rule T1 — Most tests should target Domain**

- Entities and VOs: invariants, transitions, calculations.
    

✅ **Right**

- `Order.confirm()` rejects empty order.
    
- `CPF.create()` rejects invalid.
    

❌ **Wrong**

- Testing controllers heavily but missing domain rules.
    

---

### 4.2 Integration Tests — Application + Infrastructure

**Rule T2 — Integration tests validate use cases with real implementations**

- Real DB (docker/testcontainers) or sqlite/postgres test instance
    
- Real repository implementation
    
- Verify transactions, persistence, unique constraints, mapping.
    

✅ **Right**

- `CreateOrderUseCase` creates and persists order; repo retrieves same.
    

---

### 4.3 Mocks — rules to avoid “Mocking Hell”

**Rule M1 — Mock at the edges, not at the center**

- Domain: usually no mocks.
    
- Application: mock _external_ ports only when integration is expensive/unreliable.
    
- Prefer sociable tests: real implementations of internal collaborators.
    

✅ **Right**

- Mock `PaymentGateway` in use case unit test.
    
- Use real in-memory repo for use case tests when feasible.
    

❌ **Wrong**

- Mocking 6 collaborators + verifying internal calls (“was method X called with Y”) → brittle tests.
    

---

## 5) Code Examples

### 5.1 Use Case (Application) injecting a repository and saving an Aggregate

✅ **Right**

`// application/use-cases/create-order.usecase.ts import { OrderRepository } from "../ports/order-repository"; import { Order } from "../../domain/order/order"; import { OrderItem } from "../../domain/order/order-item"; import { Money } from "../../domain/shared/money";  export type CreateOrderCommand = {   orderId: string;   items: { productId: string; quantity: number; unitPrice: number }[];   currency: "BRL" | "USD";   externalRef?: string; };  export class CreateOrderUseCase {   constructor(private readonly orders: OrderRepository) {}    async execute(cmd: CreateOrderCommand): Promise<{ orderId: string }> {     if (cmd.items.length === 0) throw new Error("INVALID_INPUT_ITEMS_REQUIRED");      if (cmd.externalRef) {       const exists = await this.orders.existsByExternalRef(cmd.externalRef);       if (exists) throw new Error("DUPLICATE_EXTERNAL_REF");     }      const order = Order.create(cmd.orderId);      for (const i of cmd.items) {       const price = Money.from(cmd.currency, i.unitPrice);       order.addItem(OrderItem.create(i.productId, i.quantity, price));     }      order.confirm();      await this.orders.save(order);     return { orderId: order.id };   } }`

❌ **Wrong (use case coupled to ORM)**

`// directly using prisma/typeorm models here -> app depends on infra`

---

### 5.2 Unit test for a business rule inside an Entity

✅ **Right (Jest)**

`import { Order } from "../../domain/order/order";  describe("Order", () => {   it("must not confirm an empty order", () => {     const order = Order.create("order-1");     expect(() => order.confirm()).toThrow("EMPTY_ORDER");   });    it("must not allow edits after confirmation", () => {     const order = Order.create("order-1");     // add an item via domain method (assume it exists and is valid)     // order.addItem(...)     order.confirm();      expect(() => {       // order.addItem(...)     }).toThrow("ORDER_NOT_EDITABLE");   }); });`

---

## A Critical Perspective (The Counterpoint)

### Risk 1: The Mapper Trap

If every read is forced through `Entity -> DTO -> ViewModel`, you’ll waste time for little benefit.

**Standard compromise (recommended):**

- **Writes (Commands):** strict DDD — Use cases build aggregates, enforce invariants, persist via repositories.
    
- **Reads (Queries):** allow basic CQRS — query models/DTO projections directly (even from SQL views) **without** reconstructing aggregates.
    

✅ **Right**

- `GetOrderSummaryQuery` returns `OrderSummaryDTO` from a read model.
    
- `CreateOrderUseCase` uses full domain.
    

❌ **Wrong**

- Rebuilding full `Order` aggregate just to list “orders table page”.
    

---

### Risk 2: Mocking Hell

Over-mocking in Application tests makes refactors painful.

**Standard compromise (recommended):**

- Put heavy logic in **Domain** and unit test it thoroughly (no mocks).
    
- For use cases, prefer **sociable tests**:
    
    - real in-memory repository (or test DB)
        
    - mock only truly external integrations (payment provider, email gateway)
        

✅ **Right**

- 80% domain unit tests
    
- A smaller set of integration tests verifying persistence & orchestration
    

❌ **Wrong**

- 200 tests that assert “method A called B called C” and break on renames.