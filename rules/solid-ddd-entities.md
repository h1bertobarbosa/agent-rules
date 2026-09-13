# SOLID, Object Calisthenics, Rich Domain Entities & Value Objects (DDD) — Technical Standardization (Practical, "Right vs Wrong")

This guide is written to be enforceable in code reviews. It defines how SOLID principles are applied pragmatically and where business rules must live, with Entities, Value Objects and Aggregates owning the behavior that protects domain invariants.

Applies when: creating or changing a domain entity, value object, aggregate, domain service or use case; deciding where a business rule belongs; replacing primitives (strings, numbers) that carry domain meaning; introducing an interface/abstraction; writing or reviewing methods and classes for size, nesting, coupling and encapsulation; reviewing code with business logic in services, controllers or repositories.

See also: Value Object concepts follow [O que são e quando utilizar Value Objects](https://www.branas.io/blog/o-que-sao-e-quando-utilizar-value-objects) (Rodrigo Branas).

---

## 1) SOLID — Applied Pragmatically

SOLID exists to make change cheap. Apply a principle when it removes a **real, present** pressure (a second implementation, a change that ripples through many files, a dependency that blocks tests) — never to anticipate hypothetical futures.

### 1.1 SRP — One reason to change

**Rule S1 — A class/module answers to one actor or concern.**
"Reason to change" means a stakeholder or concern (billing rules, document rendering, notification channel), not "does one tiny thing".

✅ **Right**

```typescript
// domain/invoice/invoice.ts — business rules for invoices
export class Invoice {
  calculateTotal(): Money { /* ... */ }
  issue(now: Date): void { /* ... */ }
}

// infrastructure/pdf/invoice-pdf-renderer.ts — presentation of the document
export class InvoicePdfRenderer implements InvoiceRenderer {
  render(invoice: InvoiceView): Buffer { /* ... */ }
}

// application/use-cases/issue-invoice.usecase.ts — orchestration
export class IssueInvoiceUseCase {
  constructor(
    private readonly invoices: InvoiceRepository,
    private readonly notifier: InvoiceNotifier,
  ) {}
}
```

❌ **Wrong**

```typescript
export class InvoiceService {
  calculateTax() { /* tax rules */ }
  generatePdf() { /* layout, fonts */ }
  sendEmail() { /* SMTP */ }
  saveToDatabase() { /* SQL */ }
}
// Tax change, layout change and SMTP change all edit the same class.
```

### 1.2 OCP — Extend by adding, not by editing

**Rule O1 — When a variation point is real and recurring, new behavior is added through a new implementation, not by editing a growing conditional.**

**Rule O2 — Do not create extension points up front.** Introduce the abstraction when the second or third variation appears.

✅ **Right** (variation already exists: several carriers with different rules)

```typescript
export interface ShippingPolicy {
  supports(carrier: CarrierCode): boolean;
  calculate(shipment: Shipment): Money;
}

export class ShippingCalculator {
  constructor(private readonly policies: ShippingPolicy[]) {}

  calculate(shipment: Shipment): Money {
    const policy = this.policies.find((p) => p.supports(shipment.carrier));
    if (!policy) throw new UnsupportedCarrierError(shipment.carrier);
    return policy.calculate(shipment);
  }
}
```

❌ **Wrong**

```typescript
calculate(shipment: Shipment): number {
  if (shipment.carrier === "CORREIOS") { /* 40 lines */ }
  else if (shipment.carrier === "JADLOG") { /* 40 lines */ }
  else if (shipment.carrier === "LOGGI") { /* 40 lines */ }
  // every new carrier edits this method and risks the others
}
```

❌ **Also wrong (premature)**

```typescript
// Only one discount rule exists, but a strategy + factory + registry was created "for the future".
export interface DiscountStrategy { /* ... */ }
export class DiscountStrategyFactory { /* ... */ }
```

### 1.3 LSP — Subtypes honor the contract

**Rule L1 — Any implementation must be usable wherever its abstraction is expected**, without the caller checking the concrete type, catching "not supported" errors, or relying on weaker guarantees.

✅ **Right** (separate contracts for different capabilities)

```typescript
export interface ReadableOrderRepository {
  findById(id: OrderId): Promise<Order | null>;
}

export interface OrderRepository extends ReadableOrderRepository {
  save(order: Order): Promise<void>;
}
```

❌ **Wrong**

```typescript
export class ArchivedOrderRepository implements OrderRepository {
  save(): Promise<void> {
    throw new Error("NOT_SUPPORTED"); // callers of OrderRepository now break at runtime
  }
}

if (repository instanceof ArchivedOrderRepository) { /* ... */ } // caller knows the subtype
```

Also a violation: an implementation that accepts less input (stricter preconditions), returns less (weaker postconditions) or throws error types the contract does not declare.

### 1.4 ISP — Small, client-specific interfaces

**Rule I1 — Ports are shaped by what the consumer needs, not by everything the provider can do.**

✅ **Right**

```typescript
export interface PaymentAuthorizer {
  authorize(payment: Payment): Promise<AuthorizationResult>;
}

export interface PaymentRefunder {
  refund(paymentId: PaymentId, amount: Money): Promise<void>;
}
```

❌ **Wrong**

```typescript
export interface PaymentGateway {
  authorize(): Promise<void>;
  capture(): Promise<void>;
  refund(): Promise<void>;
  listTransactions(): Promise<void>;
  exportReport(): Promise<void>;
  updateWebhooks(): Promise<void>;
}
// A use case that only authorizes depends on (and must fake) six methods.
```

### 1.5 DIP — Inner layers own the abstractions

**Rule D1 — Domain and Application depend on abstractions they define; Infrastructure implements them.**

**Rule D2 — Non-deterministic sources (time, randomness, ID generation) are injected**, so business rules are deterministic and testable.

✅ **Right**

```typescript
// application/ports/clock.ts
export interface Clock {
  now(): Date;
}

// application/use-cases/confirm-order.usecase.ts
export class ConfirmOrderUseCase {
  constructor(
    private readonly orders: OrderRepository,
    private readonly clock: Clock,
  ) {}

  async execute(orderId: OrderId): Promise<void> {
    const order = await this.orders.findById(orderId);
    if (!order) throw new OrderNotFoundError(orderId);
    order.confirm(this.clock.now());
    await this.orders.save(order);
  }
}
```

❌ **Wrong**

```typescript
import { PrismaClient } from "@prisma/client";

export class ConfirmOrderUseCase {
  private readonly prisma = new PrismaClient(); // concrete infra created inside

  async execute(orderId: string) {
    const order = await this.prisma.order.findUnique({ where: { id: orderId } });
    if (order.createdAt < new Date(Date.now() - 86_400_000)) { /* hidden clock */ }
  }
}
```

### 1.6 Guardrails against over-engineering

**Rule G1 — Interfaces are justified by a boundary or a real variation**, not by habit.

| Create an interface when | Do NOT create an interface for |
| --- | --- |
| It crosses a boundary: DB, HTTP, queue, cache, filesystem, clock, external API | Entities and Value Objects |
| There are, or are about to be, 2+ implementations | Pure domain services with a single implementation |
| A test double is needed for something slow or non-deterministic | DTOs, mappers, simple helpers |

**Rule G2 — Prefer a function or a small class over a pattern.** Patterns are accepted only when they reduce existing complexity (see `skills/design-patterns`).

---

## 2) Business Rules Belong to the Domain Model

### 2.1 Where each rule lives

**Rule B1 — Place a rule in the innermost element that has all the information to decide it.**

| The rule depends on… | It lives in | Example |
| --- | --- | --- |
| The meaning/format of a single value | **Value Object** (see §3) | CPF is valid; Money cannot mix currencies |
| The state of one entity or its aggregate | **Entity / Aggregate Root** | Cannot confirm an empty order; cannot add items after confirmation |
| Several aggregates, with no I/O | **Domain Service** (pure) | Price calculation from items + customer tier |
| Data that must be loaded or checked externally (uniqueness, other services, permissions) | **Use Case** (Application), which then calls the domain | External reference must be unique |
| The shape of the input (required, type, length, enum) | **API / boundary validation** | `email` is a string in email format |

**Rule B2 — Controllers, repositories, mappers, ORM hooks and database triggers never decide business rules.**

### 2.2 Tell, don't ask

**Rule B3 — Callers express intent through entity methods; they never read state, decide, and write state back.**

✅ **Right**

```typescript
order.confirm(clock.now());
```

❌ **Wrong**

```typescript
if (order.status === "DRAFT" && order.items.length > 0) {
  order.status = "CONFIRMED";
  order.confirmedAt = new Date();
}
// The rule is outside the entity; every other caller must remember to repeat it.
```

### 2.3 Encapsulation — no public mutable state

**Rule B4 — State is private. No public setters. Changes only through intention-revealing methods** (`confirm`, `cancel`, `changeShippingAddress`), never `setStatus`.

**Rule B5 — Collections are never exposed mutably.** Return a read-only view or a copy.

✅ **Right**

```typescript
export class Order {
  private readonly _items: OrderItem[] = [];

  get items(): ReadonlyArray<OrderItem> {
    return [...this._items];
  }

  addItem(item: OrderItem): void {
    this.ensureEditable();
    this._items.push(item);
  }
}
```

❌ **Wrong**

```typescript
export class Order {
  public status: string;
  public items: OrderItem[]; // order.items.push(x) bypasses every rule

  setStatus(status: string) { this.status = status; }
}
```

### 2.4 Always-valid entities

**Rule B6 — An entity cannot exist in an invalid state.** Creation goes through a named factory that enforces invariants; a private constructor prevents bypassing it.

**Rule B7 — Rehydration from persistence is separate from creation.** `restore()` rebuilds an already-valid entity without re-running creation-only rules, defaults or domain events.

✅ **Right**

```typescript
export class Order {
  private constructor(
    public readonly id: OrderId,
    private status: OrderStatus,
    private readonly _items: OrderItem[],
    private confirmedAt: Date | null,
  ) {}

  static create(id: OrderId): Order {
    return new Order(id, OrderStatus.Draft, [], null);
  }

  static restore(props: OrderSnapshot): Order {
    return new Order(props.id, props.status, props.items, props.confirmedAt);
  }
}
```

❌ **Wrong**

```typescript
const order = new Order();   // empty, invalid object
order.id = input.id;          // invariants depend on callers filling fields correctly
```

### 2.5 Explicit state transitions

**Rule B8 — Lifecycle changes are explicit methods that validate the current state.** When there are several states, declare the allowed transitions in one place.

✅ **Right**

```typescript
const ALLOWED_TRANSITIONS: Record<OrderStatus, OrderStatus[]> = {
  [OrderStatus.Draft]: [OrderStatus.Confirmed, OrderStatus.Cancelled],
  [OrderStatus.Confirmed]: [OrderStatus.Shipped, OrderStatus.Cancelled],
  [OrderStatus.Shipped]: [],
  [OrderStatus.Cancelled]: [],
};

export class Order {
  confirm(now: Date): void {
    if (this._items.length === 0) throw new EmptyOrderError(this.id);
    this.transitionTo(OrderStatus.Confirmed);
    this.confirmedAt = now;
  }

  private transitionTo(next: OrderStatus): void {
    if (!ALLOWED_TRANSITIONS[this.status].includes(next)) {
      throw new InvalidOrderTransitionError(this.id, this.status, next);
    }
    this.status = next;
  }
}
```

### 2.6 Domain errors

**Rule B9 — The domain throws typed domain errors that describe the violated rule.** No HTTP exceptions, no framework errors, no anonymous `Error("...")` for business violations. Mapping to status codes happens in the API layer.

✅ **Right**

```typescript
export class OrderNotEditableError extends DomainError {
  constructor(orderId: OrderId, status: OrderStatus) {
    super("ORDER_NOT_EDITABLE", `Order ${orderId} cannot be edited in status ${status}`);
  }
}
```

❌ **Wrong**

```typescript
throw new BadRequestException("order not editable"); // domain now depends on HTTP
throw new Error("invalid");                          // caller cannot distinguish the rule
```

### 2.7 Entities are pure

**Rule B10 — Entities and Value Objects do not perform I/O.** No repositories, HTTP clients, loggers, `async` methods, environment variables, `new Date()` or random generators inside them. Anything needed to decide is passed as an argument.

✅ **Right**

```typescript
subscription.renew(now, plan); // caller provides time and loaded data
```

❌ **Wrong**

```typescript
async renew() {
  const plan = await planRepository.findById(this.planId); // I/O inside the entity
  if (new Date() > this.expiresAt) { /* hidden clock */ }
}
```

### 2.8 Aggregates — consistency boundaries

**Rule B11 — The aggregate root is the only entry point.** Child entities are changed only through the root, which enforces the invariants that span them.

**Rule B12 — Reference other aggregates by ID, not by object.**

**Rule B13 — One aggregate is modified per transaction.** Consistency across aggregates is eventual, through domain events handled in the Application layer.

**Rule B14 — Keep aggregates small.** Include only what must be consistent immediately; a large aggregate causes lock contention and slow loads.

✅ **Right**

```typescript
export class Order {
  private readonly customerId: CustomerId;      // reference by ID
  private readonly _items: OrderItem[];

  changeItemQuantity(productId: ProductId, quantity: Quantity): void {
    this.ensureEditable();
    this.findItem(productId).changeQuantity(quantity);
    this.recalculateTotal();                     // invariant kept by the root
  }
}
```

❌ **Wrong**

```typescript
const item = await orderItemRepository.findById(itemId);
item.quantity = 10;
await orderItemRepository.save(item); // bypasses the Order root; total is now inconsistent

export class Order {
  customer: Customer; // whole aggregate embedded; loading an order loads the customer graph
}
```

### 2.9 Domain events

**Rule B15 — Entities record domain events when something meaningful happens; the Application layer dispatches them after the aggregate is persisted.** Entities never publish to a broker.

```typescript
export class Order extends AggregateRoot {
  confirm(now: Date): void {
    // ...rules
    this.record(new OrderConfirmed(this.id, now));
  }
}

// application: save first, then dispatch (or use an outbox)
await this.orders.save(order);
await this.events.publish(order.pullEvents());
```

### 2.10 Persistence stays outside

**Rule B16 — ORM/ODM models are not domain entities.** Decorators, schemas and hooks live in Infrastructure; a mapper converts between persistence models and domain objects (`restore()` / snapshot).

❌ **Wrong**

```typescript
@Entity("orders")
export class Order {
  @Column() status: string; // domain entity coupled to the ORM
}
```

---

## 3) Value Objects

A Value Object (VO) represents a domain concept defined **only by its attributes**, with no identity of its own: an e-mail, a CPF, money, a coordinate, a dimension, a period. Two VOs with the same values are the same value.

VOs are the simplest and most objective building block of a domain model: easy to model, unit-testable in isolation and highly reusable. **When starting a domain model, start with Value Objects.**

### 3.1 Primitive Obsession — the smell VOs remove

**Rule VO1 — A primitive that carries domain meaning, format or constraints must be a Value Object.**
Passing `string`, `number` or loose objects around forces every caller to remember and repeat validation, and nothing prevents an invalid value from existing.

❌ **Wrong**

```typescript
type Color = { red: number; green: number; blue: number }; // nothing keeps values within 0–255

function createCustomer(name: string, email: string, cpf: string, lat: number, long: number) {
  // Is `cpf` normalized? Is `lat` within range? Are `email` and `cpf` swapped? The type cannot tell.
}
```

✅ **Right**

```typescript
function createCustomer(name: CustomerName, email: Email, cpf: CPF, location: Coord) {
  // every argument is guaranteed valid by construction; argument order mistakes do not compile
}
```

### 3.2 Self-validating — the constructor is the single entry point

**Rule VO2 — A VO validates itself on construction and can never exist in an invalid state.**
The constructor is not only there to build the object: it is the **single entry point** that prevents an invalid instance from being created. Callers never validate a VO's data before or after creating it.

✅ **Right**

```typescript
export class Coord {
  private readonly lat: number;
  private readonly long: number;

  constructor(lat: number, long: number) {
    if (lat < -90 || lat > 90) throw new InvalidLatitudeError(lat);
    if (long < -180 || long > 180) throw new InvalidLongitudeError(long);
    this.lat = lat;
    this.long = long;
  }

  getLat(): number {
    return this.lat;
  }

  getLong(): number {
    return this.long;
  }
}
```

❌ **Wrong**

```typescript
export class Coord {
  constructor(public lat: number, public long: number) {}
}

if (isValidCoord(lat, long)) new Coord(lat, long); // validation lives outside and can be skipped
```

### 3.3 Immutable — change means a new instance

**Rule VO3 — A VO is immutable.** State is private and `readonly`; there are no setters. Once built, nothing can move it into an invalid state.

**Rule VO4 — Operations return a new instance** instead of modifying the current one. Entities replace the whole VO.

✅ **Right**

```typescript
export class Money {
  private constructor(
    private readonly amountInCents: number,
    private readonly currency: Currency,
  ) {}

  add(other: Money): Money {
    if (this.currency !== other.currency) throw new CurrencyMismatchError(this.currency, other.currency);
    return new Money(this.amountInCents + other.amountInCents, this.currency);
  }
}

customer.changeEmail(Email.create(input.email)); // entity swaps the whole VO
```

❌ **Wrong**

```typescript
price.amount += 10;             // mutates a value that may be shared by other objects
address.setZipCode("01310-100"); // partially updated VO may now be inconsistent
```

### 3.4 Equality by value

**Rule VO5 — VOs are compared by their values, never by reference or identity.** Expose an `equals()` method; do not compare with `===` or by an ID.

| | Entity | Value Object |
| --- | --- | --- |
| Identity | Has a unique ID that persists over time | None |
| Equality | Same ID → same entity, even if attributes differ | Same values → same value |
| Mutability | State changes through methods | Immutable; replaced as a whole |
| Lifecycle | Created, changed, persisted, removed | Created and discarded with its owner |

```typescript
export class Email {
  equals(other: Email): boolean {
    return this.value === other.value;
  }
}
```

### 3.5 Named factories for alternative creation paths

**Rule VO6 — When there is more than one valid way to build a VO, use a private constructor plus intention-revealing static factories.** Each factory converts to a single canonical internal representation; the private constructor keeps validation in one place. For complex construction, use a Factory or Builder.

**Rule VO7 — Normalize on creation.** Store the canonical form (unit, casing, digits only), so equality and behavior do not depend on how the value was entered.

✅ **Right**

```typescript
export class Dimension {
  private constructor(
    private readonly widthInMeters: number,
    private readonly heightInMeters: number,
    private readonly lengthInMeters: number,
  ) {
    if (widthInMeters <= 0) throw new InvalidDimensionError("width");
    if (heightInMeters <= 0) throw new InvalidDimensionError("height");
    if (lengthInMeters <= 0) throw new InvalidDimensionError("length");
  }

  static createInCentimeters(width: number, height: number, length: number): Dimension {
    return new Dimension(width / 100, height / 100, length / 100);
  }

  static createInMeters(width: number, height: number, length: number): Dimension {
    return new Dimension(width, height, length);
  }

  getVolumeInCubicMeters(): number {
    return this.widthInMeters * this.heightInMeters * this.lengthInMeters;
  }
}
```

❌ **Wrong**

```typescript
new Dimension(30, 20, 10, "cm"); // unit flag: every method must branch on it
new Dimension(30, 20, 10);       // meters or centimeters? the caller has to guess
```

### 3.6 Behavior belongs inside the VO

**Rule VO8 — Logic about the value lives in the VO:** formatting, parsing, conversion, comparison, derived values and calculations. VO methods are side-effect free.

✅ **Right**

```typescript
dimension.getVolumeInCubicMeters();
cpf.format();              // "123.456.789-09"
period.contains(date);
money.allocate([70, 30]);
```

❌ **Wrong**

```typescript
// utils/format-cpf.ts, utils/calculate-volume.ts, helpers/date-range.ts
// logic about a value scattered in helpers, duplicated across modules
```

### 3.7 When to use and when not to

**Use a Value Object when the value:**

- has validation rules or constraints (range, format, check digit, non-negative);
- needs normalization (unit, casing, masks);
- has behavior (formatting, conversion, arithmetic, comparison);
- groups attributes that only make sense together (amount + currency, lat + long, start + end);
- is repeated across entities or modules (e-mail, document, address, money).

**Do NOT create a Value Object for:**

- values with no rule, format or behavior (a free-text description, an internal flag);
- DTOs and API payloads — they carry shape, not domain meaning;
- concepts that need identity and a lifecycle — those are entities.

### 3.8 VOs at the boundaries

**Rule VO9 — Primitives are converted into VOs at the edge of the domain** (use case or mapper) and converted back when leaving it (response DTOs, persistence). Entities and domain services receive and return VOs.

**Rule VO10 — VOs are not persisted as their own tables or collections with IDs.** Store them embedded in the owner (columns, JSON/embedded document) and rebuild them in the Infrastructure mapper.

✅ **Right**

```typescript
// application
const order = Order.create(OrderId.generate(), Money.fromCents(input.totalInCents, input.currency));

// infrastructure mapper
return Order.restore({
  id: row.id,
  total: Money.fromCents(row.total_in_cents, row.currency),
});
```

❌ **Wrong**

```typescript
order.total = input.total;              // raw number enters the entity
await moneyRepository.save(order.total); // VO treated as an entity with its own table
```

---

## 4) Object Calisthenics

Object Calisthenics (Jeff Bay, *The ThoughtWorks Anthology*) is a set of 9 exercises that push code toward small objects, encapsulated behavior and low coupling. They reinforce the rest of this guide: rich entities, Value Objects, SRP and Tell, Don't Ask.

### How strictly to apply

The original rules are deliberately extreme. Apply them with the following strength:

| # | Rule | Domain layer (entities, VOs, domain services) | Application / Infrastructure / API |
| --- | --- | --- | --- |
| C1 | One level of indentation per method | **Mandatory** | Strong guideline |
| C2 | Don't use `else` | **Mandatory** | Strong guideline |
| C3 | Wrap primitives and strings | **Mandatory** (see §3) | Only at the domain boundary |
| C4 | First-class collections | **Mandatory** when the collection has rules | Guideline |
| C5 | One dot per line | **Mandatory** | Guideline |
| C6 | Don't abbreviate | **Mandatory** | **Mandatory** |
| C7 | Keep all entities small | Heuristic | Heuristic |
| C8 | Few instance variables | Heuristic | Not applicable to DTOs/config |
| C9 | No getters/setters | **No setters, no decisions from getters** | Not applicable to DTOs |

A deviation from a **Mandatory** rule must be justified in the code review. Heuristics signal a design smell to investigate, not a hard limit.

### 4.1 C1 — One level of indentation per method

**Rule C1 — A method has at most one level of indentation.** Deeper nesting means the method does more than one thing: extract a method, use guard clauses or move the logic to the object that owns the data.

✅ **Right**

```typescript
export class Order {
  confirm(now: Date): void {
    this.ensureHasItems();
    this.ensureEveryItemIsAvailable();
    this.transitionTo(OrderStatus.Confirmed);
    this.confirmedAt = now;
  }

  private ensureEveryItemIsAvailable(): void {
    for (const item of this._items) {
      item.ensureAvailable();
    }
  }
}
```

❌ **Wrong**

```typescript
confirm(now: Date): void {
  if (this.items.length > 0) {
    for (const item of this.items) {
      if (item.stock > 0) {
        if (item.quantity <= item.stock) {
          // 4 levels: validation, iteration and stock rules mixed in one method
        }
      }
    }
  }
}
```

### 4.2 C2 — Don't use `else`

**Rule C2 — Prefer guard clauses and early returns over `else`.** For variations by type or status, use polymorphism, a strategy or a lookup map (see §1.2 and §2.5).

✅ **Right**

```typescript
withdraw(amount: Money): void {
  if (this.isBlocked()) throw new AccountBlockedError(this.id);
  if (this.balance.isLessThan(amount)) throw new InsufficientBalanceError(this.id, amount);
  this.balance = this.balance.subtract(amount);
}

const FEE_BY_PLAN: Record<Plan, Percentage> = {
  [Plan.Basic]: Percentage.of(5),
  [Plan.Premium]: Percentage.of(2),
};
```

❌ **Wrong**

```typescript
withdraw(amount: number): void {
  if (!this.blocked) {
    if (this.balance >= amount) {
      this.balance -= amount;
    } else {
      throw new Error("insufficient");
    }
  } else {
    throw new Error("blocked");
  }
}
```

### 4.3 C3 — Wrap all primitives and strings

**Rule C3 — Primitives with domain meaning are Value Objects.** This is the same rule as VO1; apply everything in §3.

✅ `transfer(amount: Money, to: AccountNumber)`
❌ `transfer(amount: number, to: string)`

### 4.4 C4 — First-class collections

**Rule C4 — A collection with rules gets its own class**, and that class has no other instance variables. Filtering, totals, uniqueness and limits live in it instead of being repeated wherever the array is used.

✅ **Right**

```typescript
export class OrderItems {
  private static readonly MAX_ITEMS = 50;

  private constructor(private readonly items: ReadonlyArray<OrderItem>) {}

  static empty(): OrderItems {
    return new OrderItems([]);
  }

  add(item: OrderItem): OrderItems {
    if (this.items.length >= OrderItems.MAX_ITEMS) throw new OrderItemsLimitExceededError();
    if (this.contains(item.productId)) throw new DuplicateOrderItemError(item.productId);
    return new OrderItems([...this.items, item]);
  }

  total(currency: Currency): Money {
    return this.items.reduce((sum, item) => sum.add(item.subtotal()), Money.zero(currency));
  }

  isEmpty(): boolean {
    return this.items.length === 0;
  }

  private contains(productId: ProductId): boolean {
    return this.items.some((item) => item.productId.equals(productId));
  }
}
```

❌ **Wrong**

```typescript
// in the entity, the use case and the mapper — each one re-implements the rules
if (order.items.length >= 50) throw new Error("limit");
if (order.items.find((i) => i.productId === productId)) throw new Error("duplicate");
const total = order.items.reduce((sum, i) => sum + i.price * i.quantity, 0);
```

A plain array without rules does not need a wrapper.

### 4.5 C5 — One dot per line (Law of Demeter)

**Rule C5 — Talk only to your immediate collaborators.** Do not reach through an object graph to read or change another object's internals; ask the nearest object to do the work.

✅ **Right**

```typescript
if (order.canBeShippedTo(address)) { /* ... */ }
const city = customer.shippingCity();
```

❌ **Wrong**

```typescript
if (order.getCustomer().getAddress().getCity().getState() === "SP") { /* ... */ }
order.getItems()[0].getProduct().setStock(0);
// the caller now depends on the structure of four classes
```

Not a violation: fluent APIs and builders that return the same or a new object of the same kind (`money.add(tax).subtract(discount)`, query builders, `array.filter().map()`).

### 4.6 C6 — Don't abbreviate

**Rule C6 — Names are complete and intention-revealing.** An abbreviation usually hides a name that is too long, and a name that is too long usually means the method or class does too much.

✅ `calculateShippingFee()`, `customerRepository`, `InvoiceIssuer`
❌ `calcShipFee()`, `custRepo`, `InvIssMgr`, `tmp`, `data`, `obj`

Industry-standard, unambiguous abbreviations are allowed: `id`, `url`, `http`, `api`, `dto`, `sql`, `uuid`, `jwt`.

Repeating the class name in the method is not needed: `order.confirm()`, not `order.confirmOrder()`.

### 4.7 C7 — Keep all entities small

**Rule C7 — Classes and methods stay small enough to understand at a glance.** Use these as review signals, not mechanical limits:

| Element | Signal to split |
| --- | --- |
| Method | > 15 lines in the domain layer |
| Class | > 150 lines, or methods that use disjoint subsets of its fields |
| Module / folder | > 15 files at the same level |

A class growing past the signal usually hides a Value Object, a first-class collection or a second responsibility (§1.1).

### 4.8 C8 — Few instance variables

**Rule C8 — A class with many instance variables is a missing abstraction.** The original rule allows only two; here, when an entity or VO has more than ~4–5 fields, look for fields that change together or only make sense together and group them into a Value Object.

✅ **Right**

```typescript
export class Customer {
  private constructor(
    public readonly id: CustomerId,
    private name: PersonName,
    private contact: ContactInfo,      // email + phone
    private address: Address,          // street, number, city, state, zip code
  ) {}
}
```

❌ **Wrong**

```typescript
export class Customer {
  id: string;
  firstName: string;
  lastName: string;
  email: string;
  phone: string;
  street: string;
  number: string;
  city: string;
  state: string;
  zipCode: string;
}
```

DTOs, configuration objects and persistence models are exempt.

### 4.9 C9 — No getters and setters

**Rule C9 — Objects expose behavior, not data.** Setters are forbidden in the domain (B4). Getters are allowed for reading values to map, persist or present — but a caller must **never make a business decision from getters** and write the result back (B3).

✅ **Right**

```typescript
account.withdraw(amount);          // the account decides
invoice.isOverdue(today);          // the question is asked to the owner of the data
return { balance: account.balance().toCents() }; // getter used only to build a response
```

❌ **Wrong**

```typescript
if (account.getBalance() >= amount && !account.getBlocked()) {
  account.setBalance(account.getBalance() - amount); // rule lives outside the account
}
```

---

## 5) When a Rich Model Is Not Required

A rich domain model has a cost. Do not force it where there are no rules to protect.

- **CRUD without business rules** (reference tables, simple settings): a plain model and a thin service are acceptable.
- **Read side / queries:** return projections/DTOs directly from the read model without rehydrating aggregates — rebuilding a full aggregate just to list data adds mapping cost with no rule to protect.
- **Integration payloads and DTOs:** shape only, no behavior.

Once a business rule appears in such a module, move it into an entity or value object instead of adding it to the service.

---

## 6) Testing the Domain

- Every entity method that enforces a rule has unit tests for the allowed path and for each violation, asserting the **specific domain error**.
- Every Value Object has unit tests for valid creation, each invalid input, normalization, equality and each behavior method.
- Domain tests use **no mocks**: build objects via factories/`create()` and pass time and data as arguments.
- Test state transitions through the public methods, never by setting private fields.

```typescript
describe("Order.confirm", () => {
  it("rejects an empty order", () => {
    const order = Order.create(OrderId.generate());
    expect(() => order.confirm(FIXED_NOW)).toThrow(EmptyOrderError);
  });

  it("does not allow confirming a cancelled order", () => {
    const order = OrderFactory.cancelled();
    expect(() => order.confirm(FIXED_NOW)).toThrow(InvalidOrderTransitionError);
  });
});

describe("Dimension", () => {
  it("normalizes centimeters to meters", () => {
    const fromCentimeters = Dimension.createInCentimeters(100, 50, 20);
    const fromMeters = Dimension.createInMeters(1, 0.5, 0.2);
    expect(fromCentimeters.getVolumeInCubicMeters()).toBe(fromMeters.getVolumeInCubicMeters());
  });

  it("rejects a non-positive width", () => {
    expect(() => Dimension.createInMeters(0, 1, 1)).toThrow(InvalidDimensionError);
  });
});
```

---

## 7) Review Checklist

### SOLID

- [ ] Each class changes for one concern; unrelated responsibilities are not grouped.
- [ ] Growing `if/switch` chains over a real, recurring variation were replaced by implementations — and no abstraction was created for a variation that does not exist.
- [ ] No implementation throws "not supported" or requires type checks by callers.
- [ ] Ports are small and shaped by their consumers.
- [ ] Domain/Application depend only on abstractions they own; time, randomness and IDs are injected.
- [ ] No interfaces for entities, value objects or single-implementation pure services.

### Domain model

- [ ] Business rules live in the innermost element able to decide them (table in §2.1).
- [ ] No business decisions in controllers, repositories, mappers, ORM hooks or triggers.
- [ ] Use cases call intention-revealing methods instead of reading and assigning state.
- [ ] No public setters or mutable exposed collections.
- [ ] Entities are created through factories that enforce invariants; rehydration uses `restore()`.
- [ ] State transitions are explicit and validated.
- [ ] Domain throws typed domain errors, never HTTP/framework errors.
- [ ] Entities perform no I/O and do not read the clock or environment.
- [ ] Children are modified only through the aggregate root; other aggregates are referenced by ID; one aggregate per transaction.
- [ ] Domain events are recorded by entities and dispatched by the Application layer after persistence.
- [ ] Every rule enforced by an entity has unit tests for success and violation.

### Value Objects

- [ ] Primitives with domain meaning, format or constraints are modeled as Value Objects (no primitive obsession).
- [ ] VOs validate themselves on construction; no external validation before or after creating them.
- [ ] VOs are immutable: private `readonly` state, no setters, operations return new instances.
- [ ] VOs are compared with `equals()` by value, never by reference or ID.
- [ ] Alternative creation paths use a private constructor plus named factories, normalizing to one canonical form.
- [ ] Logic about the value (formatting, conversion, calculation) lives in the VO, not in helpers/utils.
- [ ] No VO created for values without rules or behavior, nor for DTOs.
- [ ] Primitives become VOs at the domain boundary; VOs are persisted embedded in their owner, not as tables with IDs.
- [ ] Every VO has unit tests for valid creation, invalid inputs, normalization, equality and behavior.

### Object Calisthenics

- [ ] Domain methods have at most one level of indentation; nesting was replaced by extraction or guard clauses.
- [ ] No `else` in the domain; variations use guard clauses, polymorphism or lookup maps.
- [ ] Collections with rules (limits, uniqueness, totals, filters) are first-class collections.
- [ ] No chains reaching through other objects' internals (`a.getB().getC().doSomething()`); fluent APIs excepted.
- [ ] No abbreviations beyond industry-standard ones; method names do not repeat the class name.
- [ ] Classes and methods above the size signals were justified or split.
- [ ] Entities/VOs with many fields had cohesive groups extracted into Value Objects.
- [ ] No setters in the domain; no business decisions made from getters outside the owning object.
