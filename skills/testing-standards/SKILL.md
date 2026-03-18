---
name: testing-standards
description: Enforces senior-level testing practices using Jest and Sinon, focusing on isolation, sandbox management, and the AAA pattern.
license: MIT
---

# Senior Testing Standards

This skill ensures that all test suites are robust, deterministic, and follow professional software engineering patterns. It prioritizes behavioral safety over simple line coverage.

## When to use
Apply this skill when:
- Creating new unit or integration tests.
- Refactoring existing test suites to reduce flakiness.
- Reviewing Pull Requests to ensure testing quality.
- Implementing business logic that requires high reliability.

## Core Principles

### 1. Tooling & Sandbox Strategy
- **Frameworks:** Use **Jest** as the runner and **Sinon** for mocks/stubs.
- **Cleanup:** Always use a `SinonSandbox` to prevent state leakage between tests.
- **Timers:** Use only one timer system (`jest.useFakeTimers()` OR `sinon.useFakeTimers()`). Never mix them.
 
### 2. Isolation & Structure
- **Location:** All tests must reside in the `/test` directory (never in `/src`).
- **Isolation:** Tests must not depend on each other. Each test must control its own mocks and data.
- **Pattern:** Follow **Arrange / Act / Assert (AAA)**.
- **Granularity:** Each test must verify **exactly one** behavior.

### 3. Unit vs. Integration
- **Unit Tests:** Must NOT access Network, Database, File System, or External APIs. Use Sinon stubs for all boundaries.
- **Integration Tests:** Must use real HTTP calls (e.g., `fetch`) instead of helper libraries like Supertest to validate the full network stack.

## Implementation Example

```typescript
import sinon from 'sinon';
import { MyUseCase } from './my-use-case';
import { UserFactory } from '../factories/user.factory';

describe('MyUseCase', () => {
  let sandbox: sinon.SinonSandbox;
  let repositoryStub: sinon.SinonStub;

  beforeEach(() => {
    sandbox = sinon.createSandbox();
    repositoryStub = {
      save: sandbox.stub()
    };
  });

  afterEach(() => {
    sandbox.restore();
  });

  it('should reject payment when the credit card is expired', async () => {
    // Arrange
    const expiredUser = UserFactory.build({ cardStatus: 'expired' });
    const useCase = new MyUseCase(repositoryStub);

    // Act & Assert
    await expect(useCase.execute(expiredUser))
      .rejects
      .toThrow('PaymentRejectedError');
    
    // Side Effect Assert
    sinon.assert.notCalled(repositoryStub.save);
  });
});
``` 
Please see REFERENCE.md
