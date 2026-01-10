## Node.js + TypeScript (Express & NestJS)

### Language & Tooling

- All production source code **must** be written in **TypeScript**.
    
- Use the project’s standard package manager (**npm**) and commit a single lockfile (**package-lock.json**). Do not mix package managers/lockfiles.
    
- Enable **strict type checking** (`tsconfig` with `strict: true`).
    
- Every PR must pass: `lint` + `typecheck` + `test`.
    

### TypeScript Rules

- **No `any` in production code.** Prefer:
    
    - `unknown` + type narrowing, or
        
    - proper domain types/interfaces, or
        
    - schema validation at boundaries (DTO/request payloads).
        
- If you must interact with poorly typed libraries:
    
    - add/extend typings (`@types/*` or local `*.d.ts`)
        
    - as a last resort, allow **single-line** `any` with a lint-disable comment and a short justification.
        
- Prefer **type aliases** for unions/utility types and **interfaces** for object shapes that are extended/merged.
    
- Avoid type assertions (`as X`). Use runtime checks/narrowing whenever possible.
    

### Module System & Imports

- Use **ES Modules** syntax everywhere: `import` / `export`.
    
- Never use `require` or `module.exports`.
    
- Prefer **named exports**. Use default exports only when a framework/library convention strongly benefits (e.g., React component default export) or when the module truly represents one primary thing.
    

### Variables & Mutability

- Never use `var`.
    
- Prefer `const`. Use `let` only when reassignment is required.
    
- Prefer immutable updates (especially for DTOs and pure functions).
    

### Async & Concurrency

- Prefer **async/await** for Promise-based code.
    
- Do not use `array.forEach(async ...)`. Use `for...of` (sequential) or `Promise.all` (parallel) intentionally.
    
- Never leave “floating” Promises (always `await`, `return`, or handle explicitly).
    
- Callbacks are allowed when required by Node patterns (events/streams) or third-party APIs—wrap into Promises only when it improves clarity.
    

### Collections & Iteration

- Prefer `map/filter/find/some/every` for simple transformations and predicates.
    
- Prefer `for...of` when:
    
    - you need early exit (`break/return`),
        
    - the logic is complex,
        
    - performance matters,
        
    - you’re mixing async control flow.
        

### Classes, Encapsulation & DTOs

- Use classes when behavior matters (services, domain objects). Use interfaces/types for plain data objects.
    
- Default to **private** fields and **readonly** properties.
    
- Use `public` only when it’s part of the intended API; prefer `public readonly` for exposed state.
    
- Avoid mutable shared state inside services. Keep functions pure when possible.
    

### Error Handling

- Do not swallow errors. Errors must either be:
    
    - handled and converted into a meaningful application error, or
        
    - propagated with context.
        
- Use explicit error types (custom errors or standardized error codes/messages).
    
- Validate external inputs at boundaries (HTTP requests, message queues, external APIs). TypeScript types alone are not sufficient at runtime.
    

### Architecture & Dependency Boundaries (prevents circular deps)

- Define clear module boundaries (works for both Express and Nest):
    
    - **domain** (pure business rules) must not depend on **infra** (DB, HTTP clients, frameworks).
        
    - **application/use-cases** coordinates domain + infra ports.
        
    - **infra** implements adapters (DB repositories, external clients).
        
- Avoid circular imports; enforce with lint rules.
    
- Prefer dependency inversion (interfaces/ports) at boundaries.
    

### Framework-specific (Express & NestJS)

- **Express**
    
    - Keep route handlers thin: parse/validate input → call a service/use-case → return response.
        
    - Centralize error handling via a single error middleware.
        
- **NestJS**
    
    - Use modules to enforce boundaries (avoid cross-module “reach through” imports).
        
    - Use DTOs + validation pipes for request validation.
        
    - Keep controllers thin; business logic lives in services/use-cases.
        

### Documentation & Consistency

- Keep public APIs documented (OpenAPI/Swagger where applicable).
    
- Keep naming consistent:
    
    - files/folders: `kebab-case`
        
    - classes: `PascalCase`
        
    - functions/vars: `camelCase`
        

### Minimum “done” checklist

- `npm run lint` passes
    
- `npm run typecheck` passes
    
- `npm test` passes
    
- new/changed code has types (no implicit `any`)
    
- inputs validated at boundaries
    
- no circular imports introduced