# LLD 03 — Database: Prisma, Connection Pooling, Transactions

> **Companion HLD:** [hld/03-data-stores.md](../hld/03-data-stores.md)

---

## What Is Prisma?

Prisma is an ORM (Object-Relational Mapper). It sits between your TypeScript code and PostgreSQL:

```
TypeScript Code
    │
    │  prisma.lead.findMany({ where: { status: 'OPEN' } })
    ▼
  Prisma ORM
    │
    │  SELECT * FROM "Lead" WHERE status = 'OPEN';
    ▼
 PostgreSQL
```

Without Prisma, you'd write raw SQL strings as text. Prisma gives you:
- **Type safety**: TypeScript knows the shape of every table's rows
- **Auto-completion**: your editor knows what fields exist on a Lead
- **Compile-time errors**: typo in a field name → TypeScript error, not a runtime crash
- **Migrations**: Prisma tracks schema changes and generates SQL migration files

---

## The Prisma Client Singleton

**File:** [`src/shared/prisma.ts`](../../pipeclose-backend/src/shared/prisma.ts)

```typescript
// prisma.ts (simplified)
const base = new PrismaClient({
  log: ['error', 'warn'],
  datasources: {
    db: { url: `${DATABASE_URL}?connection_limit=20&pool_timeout=30` }
  }
});

export const prisma: PrismaClient =
  process.env.NODE_ROLE === 'agent'
    ? base.$extends(createTenantScopeGuard()) as unknown as PrismaClient
    : base;
```

**Why a singleton?** Creating a `new PrismaClient()` establishes a connection pool — a set of persistent TCP connections to PostgreSQL. You don't want 50 connection pools (one per request). One pool, shared by all requests.

**The `global` trick for dev hot-reload:**
```typescript
const globalForPrisma = global as unknown as { prisma: PrismaClient };
const base = globalForPrisma.prisma || new PrismaClient(...);
if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = base;
```
When you change a file and ts-node/nodemon hot-reloads, Node.js would re-execute this file and create a *second* connection pool on top of the first, and they'd pile up. Storing the instance on `global` prevents this — hot-reload picks up the existing pool.

---

## Connection Pooling

A **connection pool** is a pre-established set of database connections kept open and ready to use. Why?

Opening a new TCP connection to PostgreSQL takes ~5-20ms (TLS handshake, authentication). If every database query opened and closed its own connection, a page that makes 10 queries would add 50-200ms just in connection overhead.

With a pool:
1. Process starts → open N connections (default: 20)
2. Query comes in → grab an idle connection from the pool
3. Run the query → return the connection to the pool (don't close it)

```
pool (20 connections):
[conn1] IDLE ──► query runs ──► IDLE
[conn2] IDLE
[conn3] BUSY (query in progress)
...
[conn20] IDLE
```

**Pool exhaustion:** If 20 concurrent queries are all running simultaneously, the 21st query must wait. `pool_timeout=30` means: wait up to 30 seconds before giving up and throwing an error.

**The right pool size:** Rule of thumb: `(2 × CPU cores) + active_io_threads`. For a 4-core server, around 10-20 connections. Too many connections actually hurt PostgreSQL (context switching overhead).

---

## Two Databases

```typescript
// Main database (all business data)
export const prisma = new PrismaClient({ datasourceUrl: DATABASE_URL });
// Pool: 20 connections, timeout: 30s

// Logs database (SystemEvent traces only)
export const logsPrisma = new LogsPrismaClient({ datasourceUrl: LOGS_DATABASE_URL });
// Pool: 5 connections, timeout: 5s
```

Why separate? If a burst of user activity generates thousands of log events per minute, those log writes could exhaust the main connection pool, making CRM queries wait. Separate pool, separate database.

The `LogsPrismaClient` is generated from a *different* Prisma schema (`prisma/logs-schema.prisma`) that only defines the `SystemEvent` table.

---

## Prisma Schema: Defining Tables

**File:** [`prisma/schema.prisma`](../../pipeclose-backend/prisma/schema.prisma) — 150KB

This single file defines every table in the main database:

```prisma
// Example: the Lead table
model Lead {
  id          Int      @id @default(autoincrement())
  name        String
  email       String?
  status      LeadStatus @default(OPEN)
  score       Float    @default(0)
  companyId   Int
  createdBy   Int
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  
  // Relations
  company     Company   @relation(fields: [companyId], references: [id])
  activities  LeadActivity[]
  history     LeadHistory[]
  
  @@index([companyId])        // index for tenant queries
  @@index([companyId, status]) // index for filtered list queries
}

enum LeadStatus {
  OPEN
  WON
  LOST
}
```

`@@index([companyId])` — creates a PostgreSQL B-tree index. When Prisma generates `WHERE companyId = 5`, PostgreSQL uses this index to find rows instantly instead of scanning the entire table.

### Migrations

When you change the schema (add a column, create a table), you run:
```bash
npx prisma migrate dev --name add_phone_to_lead
```

Prisma generates a SQL migration file in `prisma/migrations/`. In production:
```bash
npx prisma migrate deploy
```
This applies pending migrations in order. The database never runs the same migration twice (tracked in `_prisma_migrations` table).

---

## Transactions

A transaction is a group of database operations that must either ALL succeed or ALL fail. This is an atomic unit.

### Why Transactions Matter

Imagine creating a lead without a transaction:

```typescript
// BAD: no transaction
const lead = await prisma.lead.create({ data: ... });          // step 1
await prisma.leadHistory.create({ data: { leadId: lead.id } }); // step 2 — crashes!
await prisma.agentOutbox.create({ data: { leadId: lead.id } }); // step 3 — never runs
```

If step 2 crashes (disk full, DB overloaded), you have a `Lead` row with no `LeadHistory` and no `AgentOutbox`. Your data is inconsistent.

```typescript
// GOOD: transaction
await prisma.$transaction(async (tx) => {
  const lead = await tx.lead.create({ data: ... });
  await tx.leadHistory.create({ data: { leadId: lead.id } });
  await tx.agentOutbox.create({ data: { leadId: lead.id } });
  // If any of these throw, ALL three are rolled back automatically
  return lead;
});
```

With a transaction, either all three rows exist, or none do.

### The Outbox Pattern (Key Design)

This is worth understanding deeply because it appears throughout the system.

**The problem:** After creating a lead in the database, you need to enqueue a job to Redis (BullMQ). But you can't do both atomically:

```typescript
// This can fail halfway:
await prisma.lead.create({ data: ... });  // succeeds
// CRASH HERE
await bullmqQueue.add('research', { leadId }); // never runs → lead never gets AI analysis
```

**The solution — Transactional Outbox Pattern:**

Instead of writing to Redis directly, write an `agent_outbox` row to the *same database transaction* as the lead:

```typescript
await prisma.$transaction(async (tx) => {
  const lead = await tx.lead.create({ data: ... });
  // Write the "I want to enqueue this job" record to the SAME DB transaction
  await tx.agentOutbox.create({
    data: { queue: 'research', leadId: lead.id, status: 'PENDING' }
  });
  return lead;
});
// Both committed atomically. If the process crashes now,
// the outbox row is still in the DB.
```

Then, a separate process (the AgentOutbox Relay) polls the `agent_outbox` table and pushes pending rows to Redis. If the relay crashes mid-push, the row is still `PENDING` (or `IN_PROGRESS`, which the reclaim sweep resets after 5 minutes). The job will eventually be enqueued. **Durability is guaranteed by the database, not Redis.**

---

## The Tenant Scope Guard (Agent Process Only)

**File:** [`src/shared/tenantScopeGuard.ts`](../../pipeclose-backend/src/shared/tenantScopeGuard.ts)

When `NODE_ROLE=agent`, the Prisma client is wrapped with an extension:

```typescript
export const prisma =
  process.env.NODE_ROLE === 'agent'
    ? base.$extends(createTenantScopeGuard()) as PrismaClient
    : base;
```

The guard intercepts every query. If the query's `where` clause doesn't include a `companyId` filter, it **throws an error at runtime**, before the query reaches the database.

```typescript
// In the guard (simplified):
async query({ args, query }) {
  const where = args?.where;
  if (!where?.companyId) {
    throw new Error('Missing companyId in query — tenant scope violation');
  }
  return query(args);
}
```

Why only in the Agent process? The Agent process handles AI-generated data at scale and might accidentally query across tenants if a bug slips in. The web process is fine — every controller extracts `companyId` from `req.user` (which came from the JWT — always correct). But the added safety layer in the Agent process catches mistakes during development and testing.

The `as unknown as PrismaClient` cast: `$extends()` returns a new type that TypeScript considers *different* from `PrismaClient` (it tracks extensions in the type). Since the guard doesn't add or remove any methods (it's a `query`-level interceptor), the runtime object is actually the same shape. The cast is safe here.

---

## Raw SQL When Needed

Prisma's query builder covers 99% of cases. But sometimes you need raw SQL — specifically for PostgreSQL-specific features Prisma doesn't expose:

```typescript
// The outbox relay uses raw SQL for SELECT ... FOR UPDATE SKIP LOCKED
// (Prisma has no API for this)
const rows = await prisma.$queryRaw<AgentOutboxRow[]>`
  SELECT id, queue, payload, "companyId"
  FROM agent_outbox
  WHERE status = 'PENDING'
  ORDER BY "createdAt"
  LIMIT 50
  FOR UPDATE SKIP LOCKED
`;
```

`FOR UPDATE SKIP LOCKED` means: "lock these rows for my transaction, and skip any row that's already locked by another transaction." This is how the relay handles multiple relay processes in parallel — they each grab different rows without blocking each other.

---

## Next

- [04-redis-and-bullmq.md](./04-redis-and-bullmq.md) — Redis and BullMQ in depth
- [06-multi-tenancy.md](./06-multi-tenancy.md) — how tenant isolation is enforced at every layer
