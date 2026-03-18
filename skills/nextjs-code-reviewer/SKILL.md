---
name: nextjs-code-reviewer
description: Senior-level Next.js code review guide covering architecture, performance, security, and communication best practices.
---

## 1. The "Architecture First" Mental Model
Before diving into lines of code, evaluate the high-level design. Ask:
- **App Router vs. Pages**: Does this belong in the App Router or Pages? (Assuming modern App Router usage).
- **Server vs. Client Components**: Is there a healthy balance? Look for "Client Leaf" patterns where interactivity is pushed to the edges. Avoid "use client" at the top of every file.
- **Separation of Concerns**: Ensure domain logic remains decoupled from framework-specific hooks (Clean Architecture).

## 2. Next.js Specific Checklist
### A. Data Fetching & Performance
- **Server Actions**: Ensure mutations have proper Zod validation and error handling.
- **Caching**: Check for appropriate use of `revalidatePath` or `revalidateTag`.
- **Streaming**: Verify the use of `loading.tsx` or `<Suspense>` boundaries.
- **next/image**: Check for `priority` on LCP images, alt text, and defined dimensions.

### B. Security (Security by Design)
- **Environment Variables**: Sensitive keys must be in `.env` and NOT prefixed with `NEXT_PUBLIC_`.
- **Data Leakage**: Ensure large/sensitive objects aren't accidentally serialized to the client in Server Components.
- **Authorization**: Verify access control inside Server Actions or Pages, not just the UI.

### C. Routing & Metadata
- **Dynamic Routes**: Handle "not found" states using `notFound()`.
- **SEO**: Implement `generateMetadata` correctly with unique titles and descriptions.

## 3. Engineering Best Practices (The "Senior" Lens)
- **Refactoring**: Suggest breaking down PRs over 400 lines or splitting "Long Methods/Large Classes."
- **Testing Strategy**: Ensure unit tests for logic-heavy utilities and Playwright/Cypress for critical E2E flows. Test Server Actions as independent functions.

## 4. Communication Feedback (The "Senior" Behavior)
- **Tone**: Use questions (e.g., "Could we make this a Server Component?") instead of commands.
- **Context**: Explain the "Why" by linking to documentation or design principles.
- **Positive Reinforcement**: Praise clever optimizations or good habits.
- **The "Nit" Tag**: Use `nit:` for non-blocking minor preferences.

## 5. Automated Guardrails
Proactively check that the team uses:
- **ESLint**: `eslint-config-next`.
- **Prettier**: For consistent formatting.
- **TypeScript**: Strict mode enabled.
- **Lighthouse CI**: To catch performance regressions.

## 6. git
create review formal type: COMMENT 
