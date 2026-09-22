# HLD 05 — The Request Lifecycle

> **Follow a single HTTP request, step by step, from the browser to the database and back.**

This is the most important thing to understand about any web backend. We'll use "create a new lead" as the example.

---

## The Complete Journey

```
Browser
  POST /api/leads
  Body: { name: "Aditya", email: "a@example.com", companyId: 5 }
  Headers: { Authorization: "Bearer eyJhbG..." }
         │
         │ (network)
         ▼
[Load Balancer / nginx]
         │
         ▼
[Web Process — Express app]
         │
         ├──► Step 1: corsMiddleware
         ├──► Step 2: tenantMiddleware
         ├──► Step 3: tracingMiddleware
         ├──► Step 4: authMiddleware          ← route-level
         ├──► Step 5: LeadsController.create
         │              │
         │              └──► Step 6: LeadService.createLead
         │                              │
         │                              └──► Step 7: PostgreSQL INSERT
         │
         ▼
   HTTP 201 response
   { id: 99, name: "Aditya", ... }
         │
         │ (network)
         ▼
      Browser
```

---

## Step-by-Step Breakdown

### Step 0: The Browser Sends an HTTP Request

The React frontend calls `fetch('/api/leads', { method: 'POST', body: JSON.stringify(data), headers: { Authorization: 'Bearer <token>' } })`.

This is a standard HTTP POST. The token in the `Authorization` header is the JWT (JSON Web Token) that was issued when the user logged in.

---

### Step 1: CORS Middleware

**File:** [`src/shared/middleware/auth.ts` — `corsMiddleware`](../../pipeclose-backend/src/shared/middleware/auth.ts)

Before anything else, Express checks: "Is this browser allowed to talk to us?"

**CORS (Cross-Origin Resource Sharing)** is a browser security mechanism. Browsers by default block JavaScript from making requests to a different domain than the page came from. So if the frontend is at `app.pipeclose.com` and the API is at `api.pipeclose.com`, the browser would normally block the request.

The server tells the browser "it's OK" by adding `Access-Control-Allow-Origin` headers.

```typescript
// Simplified: if the request's Origin is on our allowlist, let it through
if (isOriginAllowed(origin)) {
  res.header('Access-Control-Allow-Origin', origin);
  res.header('Access-Control-Allow-Credentials', 'true');
}
```

If it's an `OPTIONS` preflight request (browser checking permissions before a POST), respond 200 immediately.

**If CORS fails:** The browser never sees the response. It blocks it. The user sees a "Network Error."

---

### Step 2: Tenant Middleware

**File:** [`src/shared/middleware/tenant.ts`](../../pipeclose-backend/src/shared/middleware/tenant.ts)

This backend is **multi-tenant**: one running instance serves many companies (acme.pipeclose.com, globex.pipeclose.com, etc.). Before auth, the server needs to know *which company* this request is for.

It reads the request's hostname:
- `acme.pipeclose.com` → extract `acme`, look up company by slug in PostgreSQL
- `crm.clientcompany.com` → custom domain, look up by exact domain

The company record is cached in Redis for 120 seconds to avoid a DB hit on every single request.

```typescript
// Look up company by subdomain
const tenantIdentifier = 'acme'; // extracted from hostname
const cacheKey = `tenant:domain:acme`;

// Check Redis first
let company = await redis.get(cacheKey); // fast!
if (!company) {
  company = await companyModel.findByDomain(tenantIdentifier); // DB query
  await redis.set(cacheKey, JSON.stringify(company), 'EX', 120);
}

req.tenant = company; // attach to request for later use
```

**Result:** `req.tenant` is now populated with the company record.

---

### Step 3: Tracing Middleware

**File:** [`src/shared/middleware/tracing.ts`](../../pipeclose-backend/src/shared/middleware/tracing.ts)

Every request gets a unique **trace ID** (a UUID). This ID is threaded through every log line, every DB query, and every event triggered by this request. Later, when debugging a bug, you can search your logs by trace ID and see the complete history of what one request did.

```typescript
const traceId = randomUUID(); // e.g. "a3b7c9d1-..."
// Store it in Node's AsyncLocalStorage — available anywhere downstream
// without passing it as a function argument
runWithTrace({ traceId, ... }, async () => {
  next(); // continue to next middleware
});
```

AsyncLocalStorage is Node's equivalent of a "thread-local" variable — data that's automatically available to any code running within the same async call chain, without you needing to pass it explicitly.

When the response finishes, this middleware writes to Prometheus: request count, duration, and status code.

---

### Step 4: Auth Middleware

**File:** [`src/shared/middleware/auth.ts` — `authMiddleware`](../../pipeclose-backend/src/shared/middleware/auth.ts)

Now we check: who is making this request?

**JWT (JSON Web Token):** When a user logs in, the server generates a signed token containing their `userId`, `companyId`, `role`, and an expiry time. This token is signed with a secret key. The browser stores it and sends it in the `Authorization: Bearer <token>` header on every request.

The auth middleware:
1. Extracts the token from `Authorization: Bearer <token>` (or a session cookie)
2. Verifies the token's signature — if anyone tampered with it, this fails
3. Checks the `tokenVersion` against the database — this lets you invalidate tokens on logout or password change without waiting for them to expire
4. Falls back to "remember me" cookie if JWT is absent/expired
5. Loads the user's RBAC permissions from the database (per-request, not stored in JWT, so permission changes take effect instantly)
6. Attaches everything to `req.user`

```typescript
req.user = {
  id: 42,
  email: 'aditya@acme.com',
  companyId: 5,
  role: 'admin',
  permissions: ['leads.create', 'deals.update', ...],
};
```

**Cross-tenant check:** If `req.tenant.id` (from step 2) doesn't match `req.user.companyId` (from the JWT), the request is blocked — a user from Company A cannot make requests through Company B's subdomain.

**If auth fails:** Returns `401 Unauthorized`.

---

### Step 5: The Controller

**File:** `src/modules/leads/controllers/LeadsController.ts`

The route definition wires a URL pattern to a controller method:
```typescript
router.post('/', authMiddleware, controller.create);
```

The controller:
1. Reads and validates the request body
2. Calls the service with the validated data
3. Handles errors and formats the HTTP response

```typescript
async create(req: AuthenticatedRequest, res: Response) {
  // 1. Validate input
  const data = LeadCreateSchema.parse(req.body);
  
  // 2. Call service (business logic)
  const lead = await this.leadService.createLead(
    req.user!.id,
    req.user!.companyId,
    data
  );
  
  // 3. Return response
  return ResponseHandler.created(res, lead);
}
```

The controller is deliberately thin. No business logic lives here. It's just "translate HTTP request into service call, translate service result into HTTP response."

---

### Step 6: The Service

**File:** [`src/modules/leads/services/leadService.ts`](../../pipeclose-backend/src/modules/leads/services/leadService.ts)

This is where business logic lives:
- Validate business rules ("a lead can't be created for a company the user doesn't belong to")
- Orchestrate multiple model calls within a transaction
- Emit domain events (for downstream processing)
- Trigger agent_outbox entries (for AI pipeline)

```typescript
async createLead(userId: number, companyId: number, data: CreateLeadData) {
  // Delegate to lifecycle service (handles the full complexity)
  return this.lifecycle.createLead(userId, companyId, data);
}
```

The lead creation then:
1. INSERTs the lead row
2. Creates an initial lead history row
3. Runs initial lead scoring (`scoreOnCreate.ts`)
4. Emits a `lead.created` event on the in-process event bus
5. Writes an `agent_outbox` row `{queue: 'research'}` — kicking off the AI pipeline

All of steps 1-5 happen in a **single database transaction**. If any step fails, all are rolled back.

---

### Step 7: The Database Write

**File:** via Prisma client from [`src/shared/prisma.ts`](../../pipeclose-backend/src/shared/prisma.ts)

The model layer is just SQL queries wrapped in functions:

```typescript
// Inside LeadModel or via Prisma directly
const lead = await prisma.lead.create({
  data: {
    name: data.name,
    email: data.email,
    companyId: companyId,
    status: 'OPEN',
    createdBy: userId,
  }
});
```

Prisma translates this into:
```sql
INSERT INTO "Lead" (name, email, "companyId", status, "createdBy")
VALUES ('Aditya', 'a@example.com', 5, 'OPEN', 42)
RETURNING *;
```

---

### The Response Path (Back Up)

After the DB write succeeds:
1. Service returns the created lead object to the controller
2. Controller calls `ResponseHandler.created(res, lead)` → sends `HTTP 201` with JSON body
3. Express writes the response to the TCP socket
4. Browser receives the response, React re-renders the UI

---

## Error Handling

What if something goes wrong at any step?

- **Validation failure** (step 5): Controller returns `400 Bad Request` with error details
- **Auth failure** (step 4): Middleware returns `401 Unauthorized`
- **Business rule violation** (step 6): Service throws an error, controller catches it, returns `422 Unprocessable Entity`
- **Database error** (step 7): Service catches it, transaction rolls back, controller returns `500 Internal Server Error`

The `ResponseHandler` class standardizes all responses so the frontend always gets a consistent JSON shape.

---

## Next

- [lld/02-http-layer.md](../lld/02-http-layer.md) — the full code-level detail of routes, controllers, services, and models
- [lld/05-auth.md](../lld/05-auth.md) — deep dive into JWT, session tokens, "remember me," and token invalidation
