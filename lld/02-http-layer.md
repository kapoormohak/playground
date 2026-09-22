# LLD 02 — The HTTP Layer: Routes, Controllers, Services, Models

> **Companion HLD:** [hld/05-request-lifecycle.md](../hld/05-request-lifecycle.md)

---

## The Four-Layer Pattern

Every module in this codebase follows the same four-layer architecture. This is not unique to Pipeclose — it's a well-established pattern in backend engineering (sometimes called "layered architecture" or "N-tier architecture"):

```
HTTP Request
     │
     ▼
┌─────────────┐
│   Route     │  "When a POST comes to /leads, call this controller method"
└─────┬───────┘
      │
      ▼
┌─────────────┐
│ Controller  │  "Parse the request, validate input, call the service, return response"
└─────┬───────┘
      │
      ▼
┌─────────────┐
│   Service   │  "Business logic: rules, orchestration, transactions, events"
└─────┬───────┘
      │
      ▼
┌─────────────┐
│    Model    │  "SQL queries: talk to the database via Prisma"
└─────────────┘
```

**Why this separation?**
- The route doesn't know about SQL
- The model doesn't know about HTTP status codes
- The service can be called from multiple places (HTTP controller, another service, a BullMQ job processor) without change
- Testing: you can unit-test the service without spinning up an HTTP server

---

## Layer 1: Routes

**Example file:** `src/modules/leads/routes/leadsRoutes.ts`

A route file maps URL patterns + HTTP methods to controller functions:

```typescript
import { Router } from 'express';
import { authMiddleware } from '../../../shared/middleware/auth';

export function createLeadsRoutes(controller: LeadsController): Router {
  const router = Router();
  
  router.get('/',          authMiddleware, controller.list);       // GET /api/leads
  router.post('/',         authMiddleware, controller.create);     // POST /api/leads
  router.get('/:id',       authMiddleware, controller.getById);   // GET /api/leads/123
  router.patch('/:id',     authMiddleware, controller.update);    // PATCH /api/leads/123
  router.delete('/:id',    authMiddleware, controller.delete);    // DELETE /api/leads/123
  
  return router;
}
```

Routes are **mounted** in `server.ts`:
```typescript
app.use('/api/leads', createLeadsRoutes(leadsController));
app.use('/api/deals', createDealsRoutes(dealsController));
// ... all other modules
```

So a request to `POST /api/leads` matches the second rule above.

### Route Parameters

`:id` in `router.get('/:id', ...)` means "match any segment here and put it in `req.params.id`." So `GET /api/leads/123` gives you `req.params.id === '123'`.

Query strings are in `req.query`: `GET /api/leads?page=2&status=OPEN` → `req.query.page === '2'`.

---

## Layer 2: Controllers

**Example file:** `src/modules/leads/controllers/LeadsController.ts`

The controller is a class. Its methods are the handlers:

```typescript
export class LeadsController {
  constructor(private leadService: LeadService) {}

  create = async (req: AuthenticatedRequest, res: Response): Promise<void> => {
    try {
      // 1. Parse and validate input
      const body = LeadCreateSchema.parse(req.body);
      //            ↑ Zod schema: throws if shape is wrong
      
      // 2. Extract auth context
      const userId = req.user!.id;
      const companyId = req.user!.companyId;
      
      // 3. Call service
      const lead = await this.leadService.createLead(userId, companyId, body);
      
      // 4. Return response
      return ResponseHandler.created(res, lead);

    } catch (err) {
      if (err instanceof ZodError) {
        return ResponseHandler.validationError(res, err.errors);
      }
      return ResponseHandler.internalError(res, err);
    }
  };
}
```

**Key rules for controllers:**
- No database queries here — that's the model's job
- No business rules here — that's the service's job
- Always extract `userId` and `companyId` from `req.user` (never from `req.body`) — you can't trust the client to tell you who they are

### Input Validation with Zod

[Zod](https://zod.dev/) is a TypeScript schema validation library. You define the shape of valid input:

```typescript
const LeadCreateSchema = z.object({
  name:       z.string().min(1).max(255),
  email:      z.string().email(),
  phone:      z.string().optional(),
  companyName: z.string().optional(),
});

type LeadCreateInput = z.infer<typeof LeadCreateSchema>;
// TypeScript type automatically derived from the schema
```

`LeadCreateSchema.parse(req.body)` either:
- Returns a strongly-typed object if valid
- Throws a `ZodError` with a detailed list of what's wrong (field name, error message)

This replaces the common pattern of `if (!req.body.name) return res.status(400).json({error: 'name required'})` — Zod does all of that for you declaratively.

### The ResponseHandler

**File:** `src/shared/responses/responses.ts`

Standardizes all HTTP responses so the frontend always gets the same shape:

```typescript
// Success responses
ResponseHandler.ok(res, data)          // 200 { success: true, data: ... }
ResponseHandler.created(res, data)     // 201 { success: true, data: ... }

// Error responses
ResponseHandler.badRequest(res, msg)   // 400 { success: false, message: ... }
ResponseHandler.unauthorized(res, msg) // 401 { success: false, message: ... }
ResponseHandler.forbidden(res, msg)    // 403 { success: false, message: ... }
ResponseHandler.notFound(res, msg)     // 404 { success: false, message: ... }
ResponseHandler.internalError(res, e)  // 500 { success: false, message: ... }
```

---

## Layer 3: Services

**Example file:** [`src/modules/leads/services/leadService.ts`](../../pipeclose-backend/src/modules/leads/services/leadService.ts)

Services are classes that hold business logic. They are constructed with their dependencies injected:

```typescript
export class LeadService {
  constructor(
    private leadModel: LeadModel,
    private historyModel: LeadHistoryModel,
    private organizationModel: OrganizationModel,
    // ... other deps
  ) {
    this.lifecycle = new LeadLifecycleService(...);
  }
}
```

**Dependency Injection:** Instead of the service creating its own dependencies (`new LeadModel()`), dependencies are passed in from the outside. This makes testing easy — in a test, you pass in a mock model instead of a real one.

### Where Business Logic Lives

The `LeadService` actually delegates most logic to `LeadLifecycleService` — because lead operations are complex enough to warrant their own file (24KB vs 24KB). This is fine — services can own other services.

Key patterns in the lifecycle service:

#### 1. Database Transactions

When multiple writes need to be atomic (all succeed or all fail):

```typescript
await prisma.$transaction(async (tx) => {
  // All queries inside use `tx` instead of `prisma`
  // If anything throws, ALL writes are rolled back
  
  const lead = await tx.lead.create({ data: leadData });
  
  await tx.leadHistory.create({
    data: { leadId: lead.id, changeType: 'CREATED', ... }
  });
  
  await tx.agentOutbox.create({
    data: { queue: 'research', leadId: lead.id, status: 'PENDING' }
  });
  
  return lead;
});
```

#### 2. Domain Events via EventBus

After a successful DB write, the service emits an event on the in-process event bus:

```typescript
import { eventBus } from '../../../infrastructure/event-bus';
eventBus.emit('lead.created', { leadId: lead.id, companyId });
```

Other parts of the system can subscribe to this event without the `LeadService` needing to know about them:

```typescript
// In some other file
eventBus.on('lead.created', async ({ leadId, companyId }) => {
  await notificationService.notify(companyId, 'New lead created');
});
```

This is the **Observer pattern** — decoupled, but within the same process.

#### 3. Lead Scoring

When a lead is created or updated, scoring runs synchronously:

```typescript
// scoreOnCreate.ts
export async function scoreOnCreate(leadId: number, companyId: number) {
  const score = await leadScoringService.calculate(leadId, companyId);
  await prisma.lead.update({
    where: { id: leadId },
    data: { score }
  });
}
```

`leadScoringService.ts` is 59KB — it's the most complex scoring engine in the system. It weighs dozens of signals (job title seniority, company size, email engagement, activity count, etc.) to produce a numeric score.

---

## Layer 4: Models

**Example file:** `src/modules/leads/models/Lead.ts`

Models are thin wrappers around Prisma queries. They exist for:
1. Reusability — the same "find lead by ID" query used in 5 places, defined once
2. Business-specific query logic (the base filtering, eager-loading of relations, etc.)

```typescript
export class LeadModel {
  async findById(id: number, companyId: number): Promise<Lead | null> {
    return prisma.lead.findFirst({
      where: { id, companyId },
      include: {
        labels: true,
        activities: { orderBy: { createdAt: 'desc' }, take: 10 },
      }
    });
  }

  async create(data: CreateLeadData): Promise<Lead> {
    return prisma.lead.create({ data });
  }
}
```

Notice `companyId` is always in the `where` clause — every query is tenant-scoped.

**Why not use Prisma directly in the service?** You can, and this codebase does in places. The model class is for reusable query logic, not a strict rule.

---

## How Modules Are Wired Together in server.ts

`server.ts` is the composition root — the place where everything is instantiated and connected:

```typescript
// Instantiate models
const leadModel = new LeadModel();
const historyModel = new LeadHistoryModel();
// ...

// Instantiate services (inject models)
const leadService = new LeadService(leadModel, historyModel, ...);

// Instantiate controllers (inject services)
const leadsController = new LeadsController(leadService);

// Register routes (inject controllers)
app.use('/api/leads', createLeadsRoutes(leadsController));
```

This is "manual dependency injection" (no framework). It's verbose but transparent — you can read `server.ts` and understand the entire object graph.

---

## Common Patterns You'll See Everywhere

### `AuthenticatedRequest`

```typescript
// src/shared/types/index.ts
export interface AuthenticatedRequest extends Request {
  user?: {
    id: number;
    email: string;
    companyId: number;
    role?: string;
    permissions?: string[];
    isSuperAdmin: boolean;
  };
  tenant?: CompanyRecord;
}
```

Every controller receives `req: AuthenticatedRequest` instead of `req: Request` — the TypeScript type tells you `req.user` and `req.tenant` are available.

### The `!` Non-null Assertion

```typescript
const userId = req.user!.id; // the ! means "trust me, this is not null"
```

After `authMiddleware`, `req.user` is guaranteed to be set (or the middleware would have returned 401). The `!` tells TypeScript "don't complain about it being potentially undefined here."

---

## Next

- [03-database.md](./03-database.md) — Prisma internals, connection pooling, transactions
- [05-auth.md](./05-auth.md) — the `authMiddleware` in detail
