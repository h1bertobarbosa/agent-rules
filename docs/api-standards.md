# **HTTP & REST API Standards**

## **1. Resource Modeling**

### Naming

- Resources must:
    
    - Be in **English**
        
    - Be **plural**
        
    - Use **kebab-case**
        
- Examples:
    
    - `/customers`
        
    - `/scheduled-events`
        
    - `/customer-invoices`
        

### Resource Hierarchies

- Use nested resources only when they represent true ownership:
    
    `/customers/:customerId/invoices /playlists/:playlistId/videos`
    
- Maximum nesting depth: **2 levels**
    
    - ✅ `/customers/:id/invoices`
        
    - ❌ `/channels/:id/playlists/:id/videos/:id/comments`
        
- Deeper relationships must use:
    
    - Query parameters, or
        
    - A top-level resource.
        

---

## **2. CRUD vs Commands**

### CRUD (pure REST)

Use standard REST verbs for entity state:

|Action|Endpoint|Method|
|---|---|---|
|Create|`/users`|POST|
|Read|`/users/:id`|GET|
|List|`/users`|GET|
|Update (partial)|`/users/:id`|PATCH|
|Delete|`/users/:id`|DELETE|

> **PUT is not used**. All updates must use PATCH.

---

### Commands (business actions)

Use POST when the operation:

- Has side effects
    
- Represents a business action
    
- Is not a simple state update
    

**Pattern**

`POST /resource/:id/action-name`

Examples:

`POST /users/:id/change-password POST /invoices/:id/cancel POST /orders/:id/refund`

Commands must:

- Always use **POST**
    
- Always use **kebab-case**
    
- Return 200, 202, or 204
    

---

## **3. Payload Format**

- All requests and responses use **JSON**
    
- Headers:
    
    `Content-Type: application/json Accept: application/json`
    
- Errors must also be JSON.
    

---

## **4. Authentication & Authorization**

Every endpoint must:

- Validate authentication (401 if missing/invalid)
    
- Validate authorization (403 or 404 based on tenancy rules)
    

In multi-tenant systems:

- If a resource exists but does not belong to the tenant → return **404**
    

---

## **5. HTTP Status Codes**

### **Success**

|Code|Meaning|
|---|---|
|200|OK (data returned)|
|201|Resource created|
|202|Accepted (async)|
|204|Success, no body|

---

### **Client Errors**

|Code|Use|
|---|---|
|400|Invalid JSON, wrong types, invalid pagination|
|401|Not authenticated|
|403|Authenticated but not allowed|
|404|Resource does not exist or not visible|
|409|Conflict (duplicate, illegal state)|
|422|Domain validation failed|

---

### **Server Errors**

|Code|Use|
|---|---|
|500|Internal crash|
|502|Downstream dependency failed|
|503|Service unavailable|
|504|Timeout|

---

## **6. Error Response Format**

All errors must follow:

`{   "error": {     "code": "VALIDATION_ERROR",     "message": "Invalid request",     "details": [       { "field": "limit", "reason": "must be <= 100" }     ],     "requestId": "abc123"   } }`

---

## **7. Pagination Standard**

### Query Parameters

`?page=1&limit=20`

|Param|Rules|
|---|---|
|page|min 1|
|limit|min 1, max 100, default 20|

Invalid params → **400**

If `page > totalPages` and `totalRecords > 0` → **400**

---

### Required Response Shape

`{   "data": [ ... ],   "meta": {     "totalRecords": 124,     "currentPage": 2,     "totalPages": 7,     "limit": 20   },   "links": {     "first": "/users?page=1&limit=20",     "prev": "/users?page=1&limit=20",     "next": "/users?page=3&limit=20",     "last": "/users?page=7&limit=20"   } }`

- `prev = null` on first page
    
- `next = null` on last page
    

---

## **8. Pagination Stability**

All paginated queries must:

- Be ordered by **one indexed field**
    
- Use deterministic ordering
    

Recommended:

`ORDER BY created_at DESC`

or

`ORDER BY id ASC`

Required index:

`CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_tablename_created_at ON tablename (created_at DESC);`

---

## **9. DTO Validation (NestJS)**

- Use:
    
    - `@IsInt`
        
    - `@Min`
        
    - `@Max`
        
    - `@Transform`
        
- Enable global validation pipe:
    

`whitelist: true, transform: true, forbidNonWhitelisted: true`

---

## **10. Partial Responses & Expansion**

Large datasets must support:

### Field selection

`/customers?fields=id,name,status`

### Relationship expansion

`/customers?include=invoices`

Rules:

- Fields not requested must not be serialized
    
- Includes must be explicitly documented
    
- Nested includes are limited to 1 level
    

---

## **11. External API Calls**

When calling external services:

- Use Axios (or NestJS HttpModule backed by Axios)
    
- Must include:
    
    - Timeouts
        
    - Retry only for idempotent requests
        
    - Backoff for 429 and 5xx
        
    - Correlation ID forwarding
        
    - Logging with PII redaction