--- Name: Test Guidelines

Description: Rigorous guidelines and standards for creating unit and integration tests in NestJS projects.

--

## What I do
I ensure that all tests follow the architectural standard defined for the project, covering everything from the choice of tools to the folder structure and naming convention.

## When to use it
- When creating new test files.
- When reviewing Pull Requests that involve tests.
- When refactoring the test directory structure.

## Implementation Rules

### 1. Stack and Commands
- **Runner/Assertions:** Joke.
- **Mocks/Stubs:** Sinon (always with `sandbox`).
- **Execution:** `test pnpm`.

### 2. Structure and Naming Conventions
- **Location:** Only in the `/test` folder (never in `/src`). - **Mirroring:** The structure of `/test` must reflect that of `/src`.
- **Naming Conventions:**
- Unit tests: `test/unit/**/*.unit.test.ts`
- Integration tests: `test/integration/**/*.integration.test.ts`
- E2E tests: `test/e2e/**/*.e2e-spec.ts`

### 3. Writing Style
- **Pattern:** AAA (Arrange, Act, Assert) or Given/When/Then.
- **Granularity:** One behavior per test. Test names should describe: `scenario + action + expected result`.
- **Dates:** Use `sinon.useFakeTimers()` for deterministic behaviors.

### 4. Unit Tests (Domain and Use Cases)
- **Forbidden:** Launching the Nest app, accessing the network, database, queues, or file system.
- **Focus:** Business rules, invariants, and use cases (main flow and approaches).
- **Gateways:** Use stubs (Sinon) for external ports.

### 5. Integration Tests (HTTP & Resources)
- **Isolation:** Truncate tables or use rollback per test.
- **HTTP:** **DO NOT use supertests**. Use `fetch` (Node 18+) against the application's real port.
- **Configuration:** `beforeAll` to start the application; `afterAll` to close connections.

### 6. Quality Assurance
- **Assertions:** Validate side effects (triggered events, persistence calls).
- **Coverage:** Minimum of **90% of lines** and **80% of agencies**.
- **Cleanup:** The `afterEach` event must execute `sandbox.restore()`.


## 7. Reference Examples

### Unit Example (Sinnon Sandbox)
```typescript
import { CreateOrderUseCase } from '../../../../src/modules/orders/create-order.use-case';
import sinon from 'sinon';

describe('CreateOrderUseCase (Unit)', () => {
  const sandbox = sinon.createSandbox();
  let sut: CreateOrderUseCase;
  let repoStub: any;

  beforeEach(() => {
    repoStub = { save: sandbox.stub() };
    sut = new CreateOrderUseCase(repoStub);
  });

  afterEach(() => sandbox.restore());

  it('deve criar pedido com sucesso', async () => {
    repoStub.save.resolves({ id: '123' });
    const result = await sut.execute({ customerId: '1' });
    expect(result.id).toBe('123');
  });
});
```
### Exemplo de Integração (Fetch API)
```typescript
import { Test } from '@nestjs/testing';
import { AppModule } from '../../../../src/app.module';

describe('Orders (Integration)', () => {
  let app: any;
  let url: string;

  beforeAll(async () => {
    const mod = await Test.createTestingModule({ imports: [AppModule] }).compile();
    app = mod.createNestApplication();
    await app.init();
    await app.listen(0);
    url = `http://localhost:${app.getHttpServer().address().port}`;
  });

  afterAll(async () => await app.close());

  it('POST /orders -> 201', async () => {
    const res = await fetch(`${url}/orders`, {
      method: 'POST',
      body: JSON.stringify({ customerId: '1' }),
      headers: { 'Content-Type': 'application/json' }
    });
    expect(res.status).toBe(201);
  });
});
```

