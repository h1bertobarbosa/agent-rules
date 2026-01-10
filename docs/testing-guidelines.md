# **Testing Guidelines**

## 1. Tooling

- **Jest** is the test runner and assertion framework.
    
- **Sinon** is used for **stubs, spies, mocks, and fakes**.
    
- Always use a **Sinon sandbox** for mocking.
    
- Restore all mocks after each test.
    

`let sandbox: sinon.SinonSandbox  beforeEach(() => {   sandbox = sinon.createSandbox() })  afterEach(() => {   sandbox.restore() })`

Use **one timer system only** (choose one for the project):

- `jest.useFakeTimers()` **or**
    
- `sinon.useFakeTimers()`
    

Never mix both.

---

## 2. Test Structure

All tests must live in `/test`  
No tests are allowed inside `/src`

`/test   /unit   /integration   /helpers   /factories`

File naming:

`*.test.ts`

---

## 3. Test Isolation

- Tests **must not depend on each other**
    
- Each test must:
    
    - Create its own data
        
    - Control its own mocks
        
    - Clean up after itself
        

No shared mutable state between tests.

---

## 4. Test Organization

Use **Arrange / Act / Assert** or **Given / When / Then**.

Each test must verify **exactly one behavior**.

Bad:

`it('does everything')`

Good:

`it('should reject payment when card is expired')`

---

## 5. Time & Randomness

Any test that depends on:

- `Date`
    
- `now()`
    
- timers
    
- random IDs
    

Must use **fake timers or mocks** to make the test deterministic.

No test may depend on real time.

---

## 6. Unit Tests (`/test/unit`)

Unit tests:

- Test **one class or use case**
    
- Must not access:
    
    - HTTP
        
    - Database
        
    - File system
        
    - Message queues
        
    - External APIs
        

All external dependencies must be **stubbed with Sinon**.

### Use Case Tests

For every use case:

You must test:

- The **main flow**
    
- At least **one error flow**
    

Use **stubs for ports and repositories**.

---

## 7. Domain Tests

Domain tests:

- Run at unit level
    
- Must not use any external resources
    
- Must cover:
    
    - All business rules
        
    - All rule branches
        
    - All meaningful variations
        

Use **table-driven tests** (`test.each`) to cover combinations.

---

## 8. Integration Tests (`/test/integration`)

Integration tests are used only when:

- Real HTTP
    
- Real database
    
- Real file system
    
- Real message queues
    
- Real external APIs  
    are involved.
    

### HTTP Endpoint Tests

Rules:

- Must not use Supertest or similar libraries
    
- Must call the server via real HTTP (e.g., `fetch`)
    
- Must test:
    
    - Main success path
        
    - At least one failure path
        
    - Status code
        
    - Error message contract
        

Do **not** test business rule variations here — that belongs to use case & domain tests.

Endpoint tests validate:

- Routing
    
- Authentication / tenant isolation
    
- Validation
    
- Error mapping
    
- Serialization contract
    

---

## 9. Test Coverage

- All **new and modified code paths** must be covered by tests.
    
- Branch coverage is mandatory for domain logic.
    
- Coverage thresholds must be enforced in CI.
    

Coverage exists to ensure **behavioral safety**, not to chase 100%.

---

## 10. Lifecycle Management

Use:

- `beforeEach` → fresh state (sandbox, factories, mocks)
    
- `beforeAll` → expensive setup (DB, server)
    
- `afterAll` → teardown
    

All external resources must be closed:

- Database connections
    
- Message brokers
    
- HTTP servers
    

No test suite may leave open handles.

---

## 11. Test Data

Do not create complex objects inline.

Use:

`/test/factories /test/helpers`

to build domain objects consistently.

---

## 12. Assertion Rules

Every test must:

- Assert the expected result
    
- Assert side effects
    
- Assert errors explicitly
    

When testing errors:

- Assert error **type**
    
- Assert error **code**
    
- Assert error **message** (when part of contract)
    

Never write tests that “pass” without checking behavior.