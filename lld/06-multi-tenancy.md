# LLD 06 — Multi-Tenancy: Three Layers of Isolation

> **Companion HLD:** [hld/04-modules-map.md](../hld/04-modules-map.md)

---

## What Is Multi-Tenancy?

Pipeclose is a SaaS product. "SaaS" means the same software instance (same running server, same database) serves multiple paying customers ("tenants"). Each customer gets their own isolated workspace.

**The hard requirement:** Company A must NEVER see Company B's leads, emails, or deals. Ever. Not even accidentally.

This isn't a business rule you implement once. It's a systemic constraint enforced at multiple layers simultaneously — because any single layer can have bugs, and the next layer is the fallback.

---

## The Three Enforcement Layers

```
Layer 1: HTTP Middleware (tenantMiddleware + authMiddleware)
         ↓ identifies the tenant from the hostname
         ↓ identifies the user from the JWT
         ↓ cross-checks: user must belong to the request's tenant
         
Layer 2: Application Code (every service query includes companyId)
         ↓ every Prisma query: WHERE companyId = req.user.companyId
         
Layer 3: Prisma Guard (Agent process only)
         ↓ runtime assertion: any query missing companyId = crash immediately
```

---

## Layer 1: HTTP Middleware

### tenantMiddleware

**File:** [`src/shared/middleware/tenant.ts`](../../pipeclose-backend/src/shared/middleware/tenant.ts)

Identifies which company this HTTP request is for, based on the **hostname** of the request.

**Subdomain routing:**
```
acme.pipeclose.com  →  extract "acme"  →  lookup company with slug="acme"
globex.pipeclose.com → extract "globex" → lookup company with slug="globex"
```

**Custom domain support:**
```
crm.globex.com → no ".pipeclose.com" suffix → treat entire domain as identifier
               → lookup company with customDomain="crm.globex.com"
```

Result: `req.tenant = { id: 5, name: "Acme Corp", isActive: true, ... }`

**Redis caching of tenant lookups:**
```typescript
const cacheKey = `tenant:domain:${tenantIdentifier}`;
let company = await redis.get(cacheKey);
if (!company) {
  company = await companyModel.findByDomain(tenantIdentifier);
  await redis.set(cacheKey, JSON.stringify(company), 'EX', 120); // 2 min TTL
}
req.tenant = company;
```

Without this cache, every single HTTP request would query the database just to figure out which company it's for. With the cache, only the first request per 120-second window hits the DB.

### authMiddleware Cross-Tenant Check

**File:** [`src/shared/middleware/auth.ts`](../../pipeclose-backend/src/shared/middleware/auth.ts)

After identifying the tenant (from hostname) and the user (from JWT), there's a critical check:

```typescript
// req.tenant was set by tenantMiddleware (from hostname)
// req.user.companyId was set by authMiddleware (from JWT)

if (req.tenant && req.tenant.id != req.user.companyId) {
  // Block: user's JWT says they belong to company 5,
  // but the hostname says this is company 7's subdomain
  return ResponseHandler.forbidden(res, 'Access denied: User does not belong to this sub-domain');
}
```

This prevents: a user who is authenticated for `acme.pipeclose.com` from making requests to `globex.pipeclose.com`. Even if they have a valid JWT, the hostname check catches the mismatch.

---

## Layer 2: Application Code

Every service method and every model query takes `companyId` as an explicit parameter and includes it in every `WHERE` clause:

```typescript
// LeadModel.findById
async findById(id: number, companyId: number) {
  return prisma.lead.findFirst({
    where: {
      id: id,
      companyId: companyId,  // ← always scoped to the tenant
    }
  });
}

// LeadService.createLead
async createLead(userId: number, companyId: number, data: CreateLeadData) {
  return prisma.lead.create({
    data: {
      ...data,
      companyId: companyId,  // ← inserted with the correct tenant
      createdBy: userId,
    }
  });
}
```

`companyId` flows from `req.user.companyId` (extracted from the verified JWT) → controller → service → model → SQL. It never comes from the request body — a client cannot fake their companyId.

**Implicit containment via Prisma relations:**

When you query `prisma.lead.findMany({ where: { companyId: 5 }, include: { activities: true } })`, the `include: { activities: true }` only pulls activities that are related to those specific leads. You can't get Company 7's activities by querying Company 5's leads — the foreign key constraint prevents orphaned relations.

---

## Layer 3: The Prisma Tenant Scope Guard

**File:** [`src/shared/tenantScopeGuard.ts`](../../pipeclose-backend/src/shared/tenantScopeGuard.ts)

This layer only runs in the **Agent** process (`NODE_ROLE=agent`). Why only there?

- The web process is protected by the HTTP middleware chain (layers 1 and 2 above)
- The agent process doesn't handle HTTP requests — it reads from BullMQ jobs and talks to external AI APIs. There's no HTTP middleware to enforce tenant scoping
- AI agents process data at scale, potentially across many leads. A subtle bug (forgetting to pass companyId) could process the wrong company's data

The guard is a Prisma **query extension** — it intercepts every Prisma query before it reaches the database:

```typescript
export function createTenantScopeGuard() {
  return Prisma.defineExtension({
    query: {
      $allModels: {
        async findMany({ args, query }) {
          if (!args?.where?.companyId) {
            throw new Error(
              `[TenantScopeGuard] Missing companyId in findMany — potential cross-tenant query`
            );
          }
          return query(args);
        },
        async findFirst({ args, query }) { /* same check */ },
        async create({ args, query }) { /* same check */ },
        async update({ args, query }) { /* same check */ },
        // etc.
      }
    }
  });
}
```

If any agent code accidentally does:
```typescript
// BUG: forgot companyId
const leads = await prisma.lead.findMany({ where: { status: 'OPEN' } });
```

The guard throws an error immediately, before any SQL runs. This catches bugs in development and in production before data ever leaks.

**The `as unknown as PrismaClient` cast:**
```typescript
export const prisma: PrismaClient = 
  process.env.NODE_ROLE === 'agent'
    ? (base.$extends(createTenantScopeGuard()) as unknown as PrismaClient)
    : base;
```

`base.$extends(...)` returns a type that TypeScript considers a *subtype* of `PrismaClient` (TypeScript knows an extension was applied). Since our extension only intercepts existing methods and doesn't add new ones, the runtime shape is identical to `PrismaClient`. The cast tells TypeScript "treat this as a plain PrismaClient" so the rest of the codebase doesn't need to know an extension is active.

---

## Multi-Tenancy in the Database Schema

Every table that stores tenant-specific data has a `companyId` column with a foreign key to the `Company` table:

```prisma
model Lead {
  id        Int     @id
  companyId Int
  company   Company @relation(fields: [companyId], references: [id])
  // ...
  
  @@index([companyId])
}
```

The `@@index([companyId])` is critical for performance. When you run `WHERE companyId = 5`, PostgreSQL uses this index to find only Company 5's rows without scanning the entire table. Without an index, every query would do a full table scan — fine for 100 rows, catastrophically slow for 10 million.

---

## Feature Flags: Kill Switches and Agent Enablement

Two additional files control per-tenant feature access:

### Kill Switches
**File:** [`src/shared/killSwitches.ts`](../../pipeclose-backend/src/shared/killSwitches.ts)

Stored in the database, read at runtime. Let you disable a feature for all tenants (global kill switch) or per-tenant:

```typescript
const isAgentEnabled = await killSwitches.check('agent.research', companyId);
if (!isAgentEnabled) {
  return; // skip agent processing for this tenant
}
```

Use case: a bug is found in the Research Agent at 2am. Instead of deploying a fix at 2am, you flip a kill switch in the database — Research Agent stops processing new jobs. Fix it in the morning, flip the switch back.

### Agent Tenant Enablement
**File:** [`src/shared/agentTenantEnablement.ts`](../../pipeclose-backend/src/shared/agentTenantEnablement.ts)

Similar, but specifically for controlling which tenants have AI features enabled. New tenants might be on a plan that doesn't include AI — the agent processes check this before doing any work for a lead.

---

## Summary: The Isolation Stack

```
Request: PATCH /api/leads/99
         from acme.pipeclose.com
         Authorization: Bearer <JWT for user in company 7>

Layer 1 — tenantMiddleware:
  req.tenant = Company { id: 5, name: "Acme Corp" }

Layer 1 — authMiddleware:
  req.user = { id: 42, companyId: 7 }
  CROSS-TENANT CHECK: req.tenant.id (5) ≠ req.user.companyId (7)
  → 403 Forbidden ← request blocked here

(Layers 2 and 3 never even run)
```

```
Request: PATCH /api/leads/99
         from acme.pipeclose.com
         Authorization: Bearer <JWT for user in company 5>

Layer 1 — tenantMiddleware:   req.tenant = Company { id: 5 }
Layer 1 — authMiddleware:     req.user = { id: 42, companyId: 5 }
  CROSS-TENANT CHECK: passes ✓

Layer 2 — LeadService:
  leadModel.findById(99, req.user.companyId)
  → WHERE id=99 AND companyId=5
  → if Lead 99 belongs to company 7, returns null → 404 Not Found ✓

(Layer 3 — Prisma guard: not active in web process, but would also catch missing companyId)
```

Three independent layers. All three must be bypassed to leak data. None of them share code paths.

---

## Next

- [07-email-system.md](./07-email-system.md) — the multi-tenant email sync system
- [05-auth.md](./05-auth.md) — JWT and session management details
