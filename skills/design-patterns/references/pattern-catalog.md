# Pattern Catalog

Read this reference only when the main `SKILL.md` requires a shortlist, trade-off reminder, or implementation sketch for a specific pattern family. Do not paste the catalog into user-facing answers; use it to choose and adapt a pattern to the repository.

## Creational Patterns

### Factory Method

Use when one product type has multiple concrete implementations and creation should be delegated to subclasses, functions, or framework-specific factories. Good fit for plugin-like construction, environment-specific implementations, and keeping callers away from concrete classes.

Trade-offs: adds an indirection point; unnecessary when a constructor is stable and visible dependencies are acceptable.

Sketch:

```ts
export interface NotificationSender {
  send(message: NotificationMessage): Promise<void>;
}

export interface NotificationSenderFactory {
  create(channel: NotificationChannel): NotificationSender;
}
```

### Abstract Factory

Use when the system must create families of related objects that must be compatible with each other, such as UI widgets per platform or provider-specific clients, mappers, and validators.

Trade-offs: useful for product families, heavy for one-off object creation.

Sketch:

```ts
export interface PaymentProviderFactory {
  createGateway(): PaymentGateway;
  createRefundService(): RefundService;
  createWebhookVerifier(): WebhookVerifier;
}
```

### Builder

Use when valid construction has many optional inputs, staged configuration, or readability problems from long parameter lists. Prefer an options object first; use Builder when staged validation or fluent composition materially improves correctness.

Trade-offs: can hide required fields if the builder is not typed or validated.

### Prototype

Use when objects are expensive to configure and new instances should be copied from a known valid prototype. Good for templates, seeded configurations, and object graphs where construction logic is already encapsulated.

Trade-offs: clone semantics must be explicit for nested mutable state; shallow copies often create subtle shared-state bugs.

### Singleton

Use rarely. Prefer dependency injection container lifecycles, module-level configuration, or explicit shared services. Only use Singleton when the single instance is part of the runtime invariant and tests can reset or replace it safely.

## Structural Patterns

### Adapter

Use when an external library, legacy module, or provider has an incompatible interface and application code should keep a stable port.

Trade-offs: mapping and error translation must be tested; avoid leaking provider types through the adapter.

Sketch:

```ts
export class StripePaymentGateway implements PaymentGateway {
  constructor(private readonly stripeClient: StripeClient) {}

  async charge(command: ChargePaymentCommand): Promise<ChargePaymentResult> {
    const response = await this.stripeClient.paymentIntents.create({
      amount: command.amountInCents,
      currency: command.currency,
    });

    return { providerId: response.id, status: response.status };
  }
}
```

### Bridge

Use when an abstraction and its implementation vary independently. Good for avoiding subclass combinations such as `EmailHtmlRenderer`, `EmailTextRenderer`, `SmsHtmlRenderer`, and `SmsTextRenderer` when channel and renderer can be composed.

Trade-offs: requires two clear axes of variation; otherwise it adds vocabulary without reducing change cost.

### Composite

Use when callers should treat leaves and groups through the same interface, such as menu trees, file trees, organization hierarchies, rule groups, or UI component structures.

Trade-offs: can blur operations that only make sense for groups; keep unsupported operations out of the shared interface when possible.

### Facade

Use when callers need a small, stable API over a complex subsystem. Good for orchestration entrypoints and simplifying repeated setup across clients.

Trade-offs: can become a god object if it accumulates unrelated responsibilities.

### Decorator

Use when adding behavior around an existing abstraction, such as caching, logging, metrics, retries, authorization checks, or validation, without changing the wrapped implementation.

Trade-offs: multiple decorators can make call flow harder to trace; keep ordering explicit.

### Flyweight

Use when a large number of objects share immutable intrinsic state and memory usage is a demonstrated concern. Good for rendering, parsing, and simulations with many repeated value combinations.

Trade-offs: separates intrinsic from extrinsic state, which can make call sites more complex; avoid without measured pressure.

### Proxy

Use when controlling access to an object: lazy initialization, remote access, authorization, caching, tracing, or rate limiting.

Trade-offs: must preserve the subject contract closely or callers will observe surprising behavior.

## Behavioral Patterns

### Strategy

Use when algorithms vary but the caller workflow is stable. Good for pricing rules, scoring, validation policies, ranking algorithms, and export formats.

Trade-offs: strategy selection needs an owner; avoid creating one class per trivial branch when a map of functions is enough.

Sketch:

```ts
export interface PricingStrategy {
  supports(customer: Customer): boolean;
  calculate(order: Order): Money;
}

export class PricingService {
  constructor(private readonly strategies: PricingStrategy[]) {}

  calculatePrice(customer: Customer, order: Order): Money {
    const strategy = this.strategies.find((candidate) => candidate.supports(customer));

    if (!strategy) {
      throw new Error("PRICING_STRATEGY_NOT_FOUND");
    }

    return strategy.calculate(order);
  }
}
```

### State

Use when an object has state-dependent behavior and conditionals are spread across methods. Each state owns allowed transitions and behavior.

Trade-offs: can create many classes; use only when states have meaningful behavior, not just labels.

### Command

Use when actions need to be represented as objects or messages for queuing, retries, audit logs, undo, scheduling, or transport across process boundaries.

Trade-offs: extra ceremony is not justified for direct synchronous method calls without lifecycle needs.

### Chain of Responsibility

Use when multiple handlers may process a request and each handler can accept, reject, or pass it along. Good for validation pipelines, middleware-like processing, and support/escalation flows.

Trade-offs: debugging can be harder if chain ordering is implicit.

### Iterator

Use when traversal should be decoupled from a collection's internal representation. Good for custom collections, paginated result traversal, tree traversal, and streaming-like APIs.

Trade-offs: premature abstraction is common; native language iteration is usually enough for arrays and simple collections.

### Mediator

Use when peer objects know too much about each other and interaction rules are better centralized in a coordinator. Good for UI coordination, workflow steps, and domain services orchestrating peer components.

Trade-offs: can become a god object; keep the mediated interaction narrow and cohesive.

### Memento

Use when state snapshots, undo, rollback, or restore points are needed without exposing internal object state. Good for editors, workflow drafts, and reversible operations.

Trade-offs: snapshots can be memory-heavy and stale; define retention and serialization rules.

### Observer and Domain Events

Use when independent reactions should happen after a domain or application event without coupling the source to every side effect.

Trade-offs: asynchronous or indirect reactions can obscure flow; document consistency expectations and failure handling.

### Template Method

Use when an inheritance hierarchy already makes sense and subclasses vary steps inside a stable algorithm skeleton.

Trade-offs: inheritance couples subclasses to the base class; prefer Strategy or composition when variation should be runtime-configurable.

### Visitor

Use when object structure is stable but operations over that structure change frequently. Good for ASTs and complex object graphs.

Trade-offs: adding new element types is expensive because every visitor changes.
