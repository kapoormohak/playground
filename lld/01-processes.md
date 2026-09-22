# LLD 01 — The Three Processes: Startup, Responsibilities, Shutdown

> **Companion HLD:** [hld/02-system-overview.md](../hld/02-system-overview.md)

---

## Why Read This?

You need to understand: when you run this backend, what exactly happens? Which code runs first? What does each process initialize? And when a process crashes or gets killed — what happens to in-flight work?

---

## Process 1: Web (`src/server.ts`)

**File:** [`src/server.ts`](../../pipeclose-backend/src/server.ts) — 1,014 lines

### Startup Sequence

```
1. Load environment variables from .env (or .env.production)
2. Security gate: refuse to boot in production with known dev SMTP passwords
3. Import all modules (this triggers module-level side effects — singletons init)
4. Create Express app
5. Mount global middleware (cors, body-parser, cookie-parser, compression)
6. Mount tenantMiddleware globally
7. Mount tracingMiddleware globally
8. Mount all route handlers (leads, deals, email, calendar, auth, ...)
9. Start all cron jobs
10. Create HTTP server from Express app
11. Create Socket.io server, attach Redis adapter
12. Listen on PORT
```

### What "Import All Modules" Does

When Node.js `require()`s a file, the top-level code in that file runs immediately. Several modules use this to set up singletons:

```typescript
// emailSyncQueueService.ts (simplified)
let instance: EmailSyncQueueService | null = null;
export function getEmailSyncQueueService() {
  if (!instance) instance = new EmailSyncQueueService(); // creates BullMQ queues
  return instance;
}
```

This pattern is called a **singleton** — one instance for the lifetime of the process. The same pattern appears for:
- Prisma client (`src/shared/prisma.ts`)
- Redis client (`src/shared/redisClient.ts`)
- Socket.io instance (`src/shared/socketInstance.ts`)
- All queue services

### The Middleware Stack Order

Order matters in Express. Middleware is executed in the order it's mounted:

```
Every request goes through:
1. corsMiddleware        → CORS headers, OPTIONS pre-flight handling
2. express.json()       → parse JSON body into req.body
3. cookieParser()       → parse cookies into req.cookies
4. compression()        → gzip response bodies
5. tenantMiddleware     → identify company from hostname → req.tenant
6. tracingMiddleware    → mint traceId, start metrics timer

Then per-route (only if route matches):
7. authMiddleware       → verify JWT/cookie → req.user + req.user.permissions
8. [optional] superAdminMiddleware → check isSuperAdmin
9. controller method    → validate body, call service, return response
```

### Cron Jobs Started in Web Process

These use `setInterval` or `node-cron` and run inside the web process:

| Function | File | Interval |
|---|---|---|
| `startTokenRefreshJob` | `cron/tokenRefresh.ts` | 30 min |
| `startEmailSyncJob` | `cron/emailSync.ts` | 5 min |
| `startFailedEmailRetryJob` | `cron/failedEmailRetry.ts` | 10 min |
| `startGmailWatchRenewalJob` | `cron/gmailWatchRenewal.ts` | Daily |
| `startTrashCleanupJob` | `cron/trashCleanup.ts` | Daily |
| `startLeadScoreDecayJob` | `cron/leadScoreDecay.ts` | Daily |
| `startOutboxReconciliationJob` | `cron/outboxReconciliation.ts` | 5 min |
| `startSendCapResetJob` | `cron/sendCapReset.ts` | Daily midnight |
| `startReprocessCron` | `cron/reprocessUnlinkedEmails.ts` | Every 5 min |
| `startActivityReminderProcessor` | `cron/activityReminderProcessor.ts` | Every min |
| `startPurchasedContactExpiryJob` | `cron/purchasedContactExpiry.ts` | Daily |
| `startSummaryJobReaper` | `cron/summaryJobReaper.ts` | 15 min |
| `startOrphanRecovery` | `cron/orphanRecovery.ts` | 15 min |

**Problem with running crons in the web process:** If you run 3 web instances behind a load balancer, each runs these crons independently. That means `leadScoreDecay` runs 3 times per day instead of once. This system doesn't currently use distributed locking for these crons — it's accepted as "mostly idempotent" for now.

---

## Process 2: Worker (`src/worker.ts`)

**File:** [`src/worker.ts`](../../pipeclose-backend/src/worker.ts) — 159 lines

### Startup Sequence

```
1. process.env.NODE_ROLE = 'worker'
2. Import dotenv (load .env)
3. Set up Socket.io with Redis adapter (emit-only, no HTTP listener)
4. Instantiate the service graph that BullMQ workers need:
   ├── EmailModel
   ├── OAuthService
   ├── EmailConnectorService
   ├── RealTimeNotificationService (wired to Socket.io)
   └── EmailService (registers itself as the singleton getEmailService() returns)
5. getEmailSyncQueueService().initWorkers() → starts 5 email BullMQ workers
6. getCalendarSyncQueueService().initWorker() → starts calendar BullMQ worker
7. getScheduledEmailWorkerService().initWorker() → starts scheduled email worker
8. startSequenceStepScheduler() → setInterval loop, every 1 min
9. startGateCalibrationCron() → node-cron, daily
10. startProposalExpiryCron() → node-cron, daily
11. Start a tiny HTTP server on WORKER_METRICS_PORT (default 4001) for Prometheus scraping
```

### Why Socket.io With No HTTP Listener?

The Worker needs to emit events to browsers (e.g., "email sync complete, refresh your inbox"). But browsers connect to the Web process, not the Worker. The solution:

```typescript
// Worker creates a Socket.io server instance but gives it NO http server
workerIO = new SocketIOServer({ transports: ['websocket'] });
workerIO.adapter(createAdapter(pubClient, subClient)); // connect via Redis
```

When `workerIO.to(userId).emit('emailSynced', data)` is called:
1. This goes to Redis pub/sub (not to a socket directly — there are none)
2. The Web process's Socket.io, subscribed to the same Redis, receives the message
3. The Web process's Socket.io finds the actual WebSocket for `userId` and delivers it

### The Service Graph Pattern

The Worker initializes services in a specific order because they depend on each other:

```
OAuthService (no deps)
    └── EmailConnectorService(oauthService)  (needs OAuth to refresh tokens)
            └── EmailService(emailModel, emailConnectorService, notificationService)
                    └── [sets the getEmailService() singleton]
                            └── [BullMQ workers call getEmailService() to process jobs]
```

If you tried to start the BullMQ workers before `new EmailService(...)` ran, workers would call `getEmailService()` and get `null` → crash.

### Graceful Shutdown

```typescript
async function shutdown(signal: string) {
  stopSequenceStepScheduler();     // stop setInterval loop
  stopGateCalibrationCron();       // stop cron
  stopProposalExpiryCron();        // stop cron
  await queueService.close();      // drain in-flight BullMQ jobs, then close
  await scheduledEmailWorker.close();
  await flushLogsBuffer();         // write any buffered trace events to DB
  process.exit(0);
}
process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT',  () => shutdown('SIGINT'));
```

`queueService.close()` tells BullMQ: "stop accepting new jobs, finish the ones already in flight, then close the Redis connection." This is how in-flight work isn't lost when you deploy a new version.

---

## Process 3: Agent (`src/agent.ts`)

**File:** [`src/agent.ts`](../../pipeclose-backend/src/agent.ts) — 139 lines

### The Critical First Line

```typescript
process.env.NODE_ROLE = 'agent'; // MUST be before any imports
```

This line sits **before all `import` statements**. That sounds weird — normally imports come first. But TypeScript compiles to CommonJS, where imports become `require()` calls at the top of the function body. The `process.env.NODE_ROLE = 'agent'` assignment gets compiled to an expression that runs before those `require()` calls.

Why does it matter? [`src/shared/prisma.ts`](../../pipeclose-backend/src/shared/prisma.ts) reads `process.env.NODE_ROLE` at module-load time (line 97-98) to decide whether to wrap the Prisma client in the tenant scope guard. If `NODE_ROLE` isn't set yet when `prisma.ts` is first `require()`d, the guard won't activate.

### Startup Sequence

```
1. process.env.NODE_ROLE = 'agent'  ← BEFORE imports
2. Import dotenv, http, logger, etc.
3. getAgentQueueRegistry()          ← creates the 6 BullMQ Queue objects in Redis
4. startAgentOutboxRelay(registry)  ← starts the 3-second polling loop
5. startExecutionConsumer()         ← BullMQ worker on agent-execution
6. startQualificationConsumer()     ← BullMQ worker on agent-qualification
7. startResearchConsumer()          ← BullMQ worker on agent-research
8. startStrategyConsumer()          ← BullMQ worker on agent-strategy
9. startIntentReplyConsumer()       ← BullMQ worker on agent-intent
10. startOutreachConsumer()         ← BullMQ worker on agent-outreach
11. startQueueDepthCollector()      ← Prometheus: track queue depths
12. Start HTTP server on AGENT_METRICS_PORT (default 4002) for Prometheus scraping
```

### The AgentQueueRegistry

**File:** [`src/modules/agent-layer/services/agentQueueRegistry.ts`](../../pipeclose-backend/src/modules/agent-layer/services/agentQueueRegistry.ts)

```typescript
export const AGENT_QUEUE_NAMES = [
  'qualification', 'research', 'strategy', 'outreach', 'execution', 'intent'
] as const;

export const AGENT_QUEUE_PREFIX = 'agent-';
// Redis keys: 'agent-qualification', 'agent-research', etc.
```

The registry:
1. Creates one Redis connection shared by all 6 queues (connection sharing is important — each BullMQ queue connection is a persistent TCP connection to Redis)
2. Creates one `Queue` object per queue name
3. Registers each queue for Prometheus depth tracking

**Default job options applied to all 6 queues:**
```typescript
defaultJobOptions: {
  attempts: 3,                           // retry up to 3 times
  backoff: { type: 'exponential', delay: 5_000 },  // 5s, 10s, 20s
  removeOnComplete: { count: 1_000 },    // keep last 1000 completed jobs
  removeOnFail:     { count: 1_000 },    // keep last 1000 failed jobs
}
```

`removeOnComplete: { count: 1000 }` instead of `true` is intentional. If you remove a completed job immediately, BullMQ's jobId-based deduplication breaks — it can't check "has this job already run?" if the record is gone.

### Graceful Shutdown

Same pattern as Worker, but stops the relay loop first (stops new jobs from being enqueued) then drains the BullMQ workers (finishes in-flight AI jobs):

```typescript
async function shutdown(signal: string) {
  stopAgentOutboxRelay();               // stop the 3s polling loop
  await stopExecutionConsumer();        // drain in-flight jobs
  await stopQualificationConsumer();
  await stopResearchConsumer();
  await stopStrategyConsumer();
  await stopIntentReplyConsumer();
  await stopOutreachConsumer();
  await registry.close();              // close all Queue connections to Redis
  process.exit(0);
}
```

---

## What Happens When a Process Crashes?

**Web process crash:** The load balancer stops routing traffic to that instance. Other instances handle requests. No data is lost (PostgreSQL has everything). The crashed process is restarted by the process manager (e.g., PM2, Kubernetes).

**Worker process crash:** In-flight BullMQ jobs that were being processed when the crash happened get marked as `stalled` by BullMQ after a `lockDuration` timeout (default ~30s). BullMQ automatically re-queues stalled jobs. Another Worker instance picks them up. Jobs may be processed twice — workers must be idempotent (running the same job twice must produce the same result, no duplicates).

**Agent process crash:** Same as Worker for the BullMQ consumers. Additionally, the AgentOutbox Relay's `reclaimStuckAgentOutboxRows` sweep handles rows that were claimed (set to `IN_PROGRESS`) but never marked `DONE` — it resets them to `PENDING` after 5 minutes so the relay picks them up again.

---

## Next

- [02-http-layer.md](./02-http-layer.md) — the route/controller/service/model pattern in code
- [03-database.md](./03-database.md) — Prisma, connection pools, transactions
