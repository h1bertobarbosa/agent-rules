# Next.js App Router — Architecture & Engineering Handbook

This document defines how we structure, build, and scale this Next.js (App Router) application.

The goals are:

- predictable architecture
    
- high performance
    
- safe data boundaries
    
- low cognitive load
    
- long-term maintainability
    

---

# 1️⃣ Core Principles

1. **Server-first**
    
    - Data fetching and business logic live on the server.
        
    - The client is only for interaction and presentation.
        
2. **Feature ownership**
    
    - Every business concept owns its own:
        
        - queries
            
        - actions
            
        - schema
            
        - UI
            
    - No global “god” modules.
        
3. **Thin routes**
    
    - `app/` only composes.
        
    - It does not contain business logic.
        
4. **URL is state**
    
    - Filters, pagination, sorting, and tabs belong in the URL.
        
5. **Explicit boundaries**
    
    - Server code must never leak into client bundles.
        

---

# 2️⃣ Project Structure

`app/   (marketing)/   (app)/     layout.tsx     dashboard/       page.tsx     customers/       page.tsx  features/   customers/     components/       CustomersTable.tsx       CustomersTableClient.tsx       CustomerForm.tsx     hooks/       useCustomerFilters.ts       useOptimisticCustomers.ts     server/       actions.ts       queries.ts     schema.ts     types.ts  components/   ui/         # Buttons, Inputs, Modals (no business logic)   shared/     # Sidebar, Topbar, Breadcrumbs  lib/   db/         # database client, transactions   auth/       # getAuth(), role checks   validation/   cache/   observability/`

---

# 3️⃣ Layer Responsibilities

## UI Layer (`components/ui`)

- Pure presentational components
    
- No business logic
    
- No data fetching
    
- No side effects
    

## Shared Layer (`components/shared`)

- Cross-feature layout pieces
    
- Can be server or client
    
- Minimal logic
    

## Feature Layer (`features/<name>`)

Everything business-related lives here:

- schema
    
- types
    
- queries
    
- actions
    
- UI
    
- hooks
    

## App Layer (`app/(app)`)

- Routes only
    
- Fetch data
    
- Render feature components
    
- No business logic
    

---

# 4️⃣ Server vs Client Rules

### Server by default

A file is server unless it has:

`"use client"`

Use client components only when you need:

- browser APIs
    
- events (`onClick`)
    
- React state
    
- animations
    
- optimistic UI
    

Never mark pages or layouts as client.

Use **small client islands**.

---

# 5️⃣ Server Code Boundaries

Every file in:

`features/*/server/`

must start with:

`import "server-only"`

This guarantees:

- database code never ships to the browser
    
- Next.js enforces the boundary
    

---

# 6️⃣ Data Access Pattern

## `queries.ts` (server-only)

- Read and write the database
    
- Accept `tenantId`, never read auth internally
    
- No validation, no permissions
    

`export async function listCustomers({ tenantId, page }) {}`

## `actions.ts` (server-only)

- Auth
    
- Validation
    
- Call queries
    
- Invalidate cache
    

``export async function createCustomer(input) {   const auth = getAuth()   const data = CustomerCreateSchema.parse(input)    await db.createCustomer(auth.tenantId, data)    revalidateTag(`tenant:${auth.tenantId}:customers`) }``

---

# 7️⃣ Server Actions vs API Routes

Use **Server Actions** when:

- Called from Next.js UI
    
- Internal business logic
    
- Need type safety
    

Use **API Routes (`app/api`)** when:

- Used by mobile apps
    
- Webhooks
    
- External systems
    
- Need versioning or REST semantics
    

---

# 8️⃣ Cache Strategy

### Tag naming

`tenant:{tenantId}:customers customer:{customerId}`

### Invalidate only in server actions

Never in the UI.

---

# 9️⃣ Validation

Every mutation is validated twice:

- Client → UX
    
- Server → security
    

Pattern:

`schema.ts   CustomerCreateSchema   CustomerUpdateSchema  types.ts   CustomerCreateInput   CustomerUpdateInput`

---

# 🔟 URL-Driven State

Filters, sorting, pagination belong in the URL.

Each feature owns its own hooks:

- `useCustomerFilters`
    
- `usePagination`
    
- `useSort`
    

These hooks:

- parse query params
    
- validate with Zod
    
- update router
    

---

# 1️⃣1️⃣ Forms

Pattern:

Client Form → Server Action

On success:

- toast
    
- `router.refresh()` or navigation
    

Never fetch data with `useEffect`.

---

# 1️⃣2️⃣ Optimistic UI

Use per-feature hooks:

`useOptimisticCustomers(initial)`

- Update immediately
    
- Revalidate after action
    

Never global optimistic state.

---

# 1️⃣3️⃣ Error Handling

Every action returns structured errors:

`{   code,   message,   requestId,   fieldErrors? }`

Errors are:

- logged
    
- observable
    
- user-friendly
    

---

# 1️⃣4️⃣ Pagination & Ordering

Always stable:

`ORDER BY created_at DESC, id DESC`

No unstable lists.

---

# 1️⃣5️⃣ What We Never Do

🚫 Client-side fetching for primary data  
🚫 Business logic in `app/`  
🚫 Global god hooks  
🚫 Importing server code into client  
🚫 Fetching without tenant scoping  
🚫 Mutations without validation  
🚫 Cache invalidation in UI