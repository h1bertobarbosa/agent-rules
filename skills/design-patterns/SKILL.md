---
name: design-patterns
description: Choose, explain, and implement software design patterns in real codebases. Use when users ask "which pattern should I use", "apply Strategy/Factory/Adapter", "reduce conditionals", "decouple this module", "make this easier to extend/test", or request architecture/refactoring with recurring object-collaboration problems. Do NOT use for generic code cleanup, framework-specific best practices, or domain boundary analysis unless a design pattern decision is central.
license: CC-BY-4.0
metadata:
  author: agent-rules maintainers
  version: "1.0.0"
---

# Design Patterns

Use this skill to select and apply design patterns pragmatically. The goal is not to force GoF patterns into code; it is to improve extensibility, testability, dependency direction, and clarity with the smallest useful design change.

## Instructions

### Step 1: Diagnose the Real Design Pressure

Inspect the current code or ask for the missing context needed to identify the pressure behind the request. Look for repeated conditionals, unstable construction logic, awkward dependencies, hard-to-test collaborators, framework or infrastructure leakage, duplicated algorithms, or changes that require edits across many files.

Expected output: a short diagnosis that names the concrete pain, the affected files or modules, and whether a design pattern is warranted.

### Step 2: Classify the Problem

Classify the pressure before choosing a pattern:

- **Creational**: object construction varies by type, policy, environment, or family of related implementations.
- **Structural**: existing objects need compatible interfaces, layered boundaries, composition, decoration, or controlled access.
- **Behavioral**: algorithms, state transitions, notifications, commands, or responsibility chains need to vary independently from callers.

If the user asks for a named pattern, still verify that the pressure matches the pattern. If it does not, state the mismatch and recommend the simpler fit.

### Step 3: Compare Options

Offer 1-2 viable options, not a catalog dump. Include the recommended option, the rejected option, and the reason in terms of coupling, volatility, testability, API clarity, and migration cost.

Read `references/pattern-catalog.md` only when you need a pattern shortlist, trade-off reminder, or implementation sketch for a specific pattern family.

Expected output: a concise recommendation that explains why the chosen pattern solves this repository's problem.

### Step 4: Design the Roles

Name the roles before editing code: client, abstraction, concrete implementation, factory, strategy, adapter, handler, state, event subject, or other collaborators. Keep names domain-specific first and pattern-specific second, for example `PaymentPricingStrategy` is usually clearer than `ConcreteStrategyA`.

For Clean Architecture codebases, keep dependency direction inward. Put ports or abstractions in the application or domain layer when inner code needs to depend on behavior, and keep framework, database, HTTP, filesystem, time, randomness, and environment access in infrastructure adapters.

Expected output: a role map tied to actual files or proposed files.

### Step 5: Implement Incrementally

Make the smallest production-oriented change that proves the pattern. Preserve public contracts unless the user explicitly allows breaking changes. Prefer composition and explicit dependencies over global state. Avoid Singleton unless the runtime or framework lifecycle already owns the single instance.

When refactoring existing code, keep behavior equivalent first, then improve naming or extraction only where it supports the pattern.

Expected output: scoped code changes with clear boundaries and no unrelated refactors.

### Step 6: Verify Behavior

Add or update focused tests when the pattern changes behavior, moves branching logic, introduces abstractions, or affects shared code. At minimum, provide executable verification steps. Include one test for the stable caller API and one test for a new variation when the value of the pattern is extensibility.

Expected output: passing tests or a clear note explaining which verification could not be run.

## Decision Checklist

- Is this a recurring variation point rather than a one-off?
- Does the design reduce caller knowledge or just add indirection?
- Is the abstraction owned by the stable side of the dependency?
- Can a new variant be added with fewer risky edits?
- Is the resulting API easier to test and read?
- Are lifecycle, performance, and memory costs acceptable?

## Pattern Selection Guide

- Use **Factory Method** or **Abstract Factory** when creation policy varies and callers should not know concrete classes.
- Use **Builder** when constructing a valid object requires multiple optional or staged inputs.
- Use **Prototype** when cloning configured objects is safer than rebuilding them from scratch.
- Use **Adapter** when integrating incompatible interfaces without changing either side.
- Use **Bridge** when abstraction and implementation vary independently and subclass combinations are multiplying.
- Use **Composite** when callers should treat individual objects and object groups uniformly.
- Use **Facade** when callers need a smaller API over a complex subsystem.
- Use **Decorator** when behavior should be added around an object without subclass explosion.
- Use **Flyweight** when many small objects share immutable intrinsic state and memory pressure is real.
- Use **Proxy** when access needs lazy loading, caching, authorization, remote calls, or instrumentation.
- Use **Strategy** when algorithms vary behind the same caller workflow.
- Use **State** when object behavior changes according to internal state and conditionals are growing.
- Use **Command** when requests need queuing, undo, audit, retries, or transport across boundaries.
- Use **Chain of Responsibility** when multiple handlers may process or pass on a request.
- Use **Iterator** when collection traversal should be exposed without leaking internal representation.
- Use **Mediator** when peer objects are tightly coupled through direct references and interaction rules belong in one coordinator.
- Use **Memento** when state snapshots, rollback, or undo are required without exposing internal state.
- Use **Observer** or domain events when independent reactions should follow a change without coupling the source to subscribers.
- Use **Template Method** only when inheritance is already appropriate and the algorithm skeleton is stable.
- Use **Visitor** only when operations vary more often than the object structure and double dispatch is worth the complexity.

## Output Format

When applying this skill, respond with:

1. **Diagnosis**: the design pressure and why a pattern is or is not justified.
2. **Recommendation**: selected pattern, alternatives considered, and trade-offs.
3. **Role map**: abstractions and concrete collaborators tied to files.
4. **Implementation**: code changes or a precise implementation plan.
5. **Verification**: tests run, tests added, or manual verification.
6. **Risks**: migration, rollback, and over-engineering risks.

For small requests, compress this format into a few focused paragraphs, but keep the same reasoning.

## Examples

### Example 1: Replace Type Conditionals

User says: "This pricing service has a big switch by customer type. Apply a pattern."

Actions:

1. Inspect the switch and callers.
2. Recommend Strategy if algorithms vary by customer type and share one pricing contract.
3. Define `PricingStrategy`, concrete strategies, and a resolver/factory if selection is external.
4. Add tests proving each strategy and the caller workflow.

Result: the caller depends on `PricingStrategy`, new customer types add a strategy instead of editing a large switch.

### Example 2: Integrate an External API

User says: "This payment provider has a different interface. Should I use Adapter?"

Actions:

1. Compare the provider API with the application's expected payment port.
2. Recommend Adapter when the domain/application contract is already stable.
3. Keep provider SDK details in infrastructure.
4. Test the adapter mapping and error translation.

Result: application code keeps using its existing port while infrastructure translates provider-specific calls.

### Example 3: Avoid Over-Engineering

User says: "Create a factory for this one class constructor."

Actions:

1. Check whether construction actually varies or hides infrastructure details.
2. Recommend a simple helper or direct constructor if there is no variation point.
3. Explain what future signal would justify a factory.

Result: no pattern is introduced when it would only add indirection.

## Troubleshooting

### The requested pattern does not fit

Cause: the user named a pattern before the design pressure was clear.

Solution: explain the mismatch, recommend the smallest fitting design, and still provide the named pattern only if the user explicitly wants it after seeing the trade-off.

### The pattern crosses architecture boundaries

Cause: abstractions or implementations were placed in layers that invert the intended dependency direction.

Solution: move the abstraction to the stable inner layer and keep concrete infrastructure details in adapters.

### The design adds too many files

Cause: the implementation copied textbook roles instead of repository-specific roles.

Solution: collapse roles that do not vary independently and keep only the abstraction points required by current or near-term variation.
