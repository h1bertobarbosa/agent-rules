---
description: Performs an automated architectural and coverage review of test files to ensure compliance with Senior Engineering Standards and a minimum of 80% logic coverage.
agent: build
---
## Skill Mapping

- **Required Skill:** `testing-standards`
    
- **Reference Document:** `REFERENCE.md`
    

## Instructions for the Agent

When this command is invoked, perform the following sequence:

### 1. File Discovery & Mapping

- Identify the source file(s) associated with the test files in the `target_path`.
    
- If no path is provided, scan the `/src` and `/test` directories to map implementation to tests.
    

### 2. The "80% Logic Coverage" Audit

- **Analyze the Source:** Identify all exported functions, conditional branches (`if/else`), and error-handling blocks (`try/catch`) in the source file.
    
- **Trace the Tests:** Map the `it()` blocks in the test file to these source components.
    
- **Enforcement:** - Verify that at least **80% of the logical branches** are exercised by a corresponding test case.
    
    - Specifically check for "The Unhappy Path": ensure error blocks are not ignored.
        
- **Failure Condition:** If coverage is estimated below 80%, identify the specific "blind spots" (e.g., _"The 'Insufficient Funds' branch in `payment-service.ts` is not covered"_).
    

### 3. Senior Sandbox & Boundary Validation

- **Sandbox:** Confirm `sinon.createSandbox()` and `sandbox.restore()` are correctly implemented in the lifecycle hooks.
    
- **Boundaries:** Flag any prohibited I/O (Database, Network) in `/test/unit`.
    
- **AAA Pattern:** Ensure the **Arrange / Act / Assert** structure is followed.
    

### 4. Reporting Output

Provide a structured report including:

- **Coverage Score:** Estimated percentage based on branch analysis.
    
- **Violations:** List of missing test cases or architectural flaws.
    
- **Refactoring Proposals:** Practical code snippets to reach the 80% threshold.
