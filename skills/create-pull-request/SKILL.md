---
name: create-pull-request 
description: Automation and standardization in the creation of Pull Requests, ensuring domain validation, test coverage with Jest/Sinon, and adherence to Clean Architecture.
---

# Create Pull Request

## 1. What I Do

- **Template Orchestration:** I load and apply local reference templates (`feature.md`, `bug.md`, `release.md`) located within the skill's folder.
    
- **Domain Invariant Validation:** I ensure the PR description explicitly maps changes to Domain Entities and Value Objects.
    
- **Quality Checklist:** I enforce the use of `sinon.sandbox` for unit tests and Fetch API for integration tests, rejecting `supertest`.
    
- **Architecture Mapping:** I categorize changes into **Core (Domain/Application)** and **Infrastructure (Adapters/Configuration)**.
    

## 2. When to Use Me

- When opening any PR to ensure the description isn't "anemic" (lacking business context).
    
- To standardize communication between developers and Senior/Staff reviewers.
    
- When moving code through the lifecycle (from development to release).
    

## 3. Rules and Patterns (Mandatory)

|**Category**|**Right (Do)**|**Wrong (Don't)**|
|---|---|---|
|**Templates**|Use the files in `./templates/` relative to the skill.|Hardcode generic descriptions in the prompt.|
|**Testing**|Mock Ports/Interfaces using `sinon.sandbox`.|Use real implementations or global mocks.|
|**Entities**|Validate business rules in the Entity constructor.|Rely only on DTO/Controller validation.|
|**I/O**|Use Fetch API for route integration tests.|Use `supertest` or heavy external libraries.|

## 4. Local References (Files in Skill Folder)

I process the following files located in the `./templates/` sub-directory:

 - `./templates/feature.md`
 - `./templates/bug.md`
 - `./templates/release.md`

## 5. Critical Perspective (Trade-offs)

- **Local Encapsulation:** Keeping templates inside the skill folder ensures portability but requires the skill itself to be updated across repositories to sync changes.
    
- **Verbosity vs. Value:** Detailed templates can feel heavy for trivial changes. _Counter-measure:_ I should skip domain sections if the diff only contains non-functional refactoring (e.g., linting).
    
- **Testing Rigor:** Enforcing `Sinon` sandboxes prevents state leaks but has a steeper learning curve than simple Jest mocks.
