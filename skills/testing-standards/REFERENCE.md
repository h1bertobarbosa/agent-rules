# Engineering Reference: Testing Strategy & Rationale

This document provides the technical depth and strategic reasoning behind the `testing-standards` skill. It serves as a guide for engineers to understand the "why" behind our constraints.

---

## 1. The Sandbox Pattern (Sinon)
**Rationale:** In high-scale Node.js environments, global state leakage is the primary cause of non-deterministic tests (flakiness).
- **The Problem:** If `stubA` is created in Test 1 and not restored, Test 2 might inadvertently use that stub instead of the real implementation, leading to false positives.
- **The Solution:** The `SinonSandbox` creates a dedicated registry for all fakes. Calling `sandbox.restore()` in the `afterEach` hook ensures that every test begins with a 100% clean environment.

## 2. Deterministic Testing (Time & Randomness)
**Rationale:** Tests must be mathematical functions: given Input A, they must always produce Output B, regardless of when or where they run.
- **Fake Timers:** Using `Date.now()` or `setTimeout` makes tests dependent on the CPU speed of the CI runner. We use `useFakeTimers()` to "freeze" or "fast-forward" time manually.
- **Randomness:** Any ID generation (UUIDs/NanoIDs) should be stubbed to return a static string (e.g., `'fixed-id-123'`) so assertions can be exact.

## 3. The Boundary Rule (Unit vs. Integration)
Based on **Clean Architecture** principles, the "Business Logic" (Domain/Use Cases) must be independent of "Details" (DB/HTTP).

### Unit Tests (The "Inner Circle")
- **Speed:** Should execute ~100 tests per second.
- **Isolation:** We stub **Ports** (Interfaces). We do not test the Database; we test how our code *interacts* with the Database interface.
- **Coverage:** This is where 100% of the logical branches (if/else, switch, catch) must be validated.

### Integration Tests (The "Outer Circle")
- **The "Fetch" Rule:** We avoid `Supertest` because it mocks the HTTP layer. By using a real `fetch` against a running local server, we validate:
    - Middleware execution (CORS, Auth, Compression).
    - Header parsing.
    - JSON Serialization/Deserialization.
- **Scope:** Integration tests should only verify the "Happy Path" and "Contract Failures" (400, 401, 404). They should NOT re-test complex business rules already covered in unit tests.

## 4. Test Data Management (Factories)
**Rationale:** Inline object creation leads to "Fragile Tests." If you add a required field to a `User` entity, you shouldn't have to fix 500 tests.
- **Factories:** Centralize object creation in `/test/factories`.
- **Minimalism:** Only define the attributes relevant to the specific test scenario. Let the factory handle the defaults for the rest.

## 5. Assertion Depth
A senior-level test does not just check if a function "didn't crash."
- **State Assertion:** Check the return value.
- **Behavior Assertion:** Use `sinon.assert.calledOnceWith` to ensure that a side-effect (like sending an email or saving to a log) happened with the exact expected payload.
- **Error Assertion:** Always catch and inspect the error object. A test that passes on *any* error is dangerous; it must pass on the *specific* error expected.

---

## Recommended Reading (Project Context)
- **Refactoring (Fowler):** Chapter on "Setting up Tests."
- **Clean Architecture (Martin):** Chapter on "The Test Boundary."
- **DDD (Evans):** On "Isolated Domain Logic."
