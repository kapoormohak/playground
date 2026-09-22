# LLD 10 — Observability: Metrics, Tracing, Logging, Audit

> **Companion HLD:** [hld/02-system-overview.md](../hld/02-system-overview.md)  
> **Directory:** [`src/shared/tracing/`](../../pipeclose-backend/src/shared/tracing) and [`src/shared/metrics/`](../../pipeclose-backend/src/shared/metrics)

---

## What Is Observability?

"Can I understand what my system is doing right now, and what it did in the past?"

Observability answers three questions:
1. **What's happening now?** — Metrics (Prometheus)
2. **Why did this request fail?** — Tracing (trace context + event log)
3. **Who did what?** — Audit log

Without observability, debugging production issues is guesswork. With it, you can answer: "At 3:47pm, User #42 called PATCH /leads/99 from company 5, it took 432ms, it wrote to the Lead table, and then agent_outbox was updated."

---

## 1. Prometheus Metrics

**[Prometheus](https://prometheus.io/)** is a time-series metrics database. Your backend exposes an HTTP endpoint (`/metrics`) that Prometheus scrapes every 15 seconds. A visualization tool like Grafana then graphs those metrics over time.

### Three Scrapers (Three Processes = Three Metric Endpoints)

| Process | Port | Endpoint |
|---|---|---|
| Web | `PORT` (3000) | `http://server:3000/metrics` |
| Worker | `WORKER_METRICS_PORT` (4001) | `http://worker:4001/metrics` |
| Agent | `AGENT_METRICS_PORT` (4002) | `http://agent:4002/metrics` |

Each process exposes its own metrics. Prometheus scrapes all three. Grafana queries Prometheus.

### Metric Types

**File:** [`src/shared/metrics/registry.ts`](../../pipeclose-backend/src/shared/metrics/registry.ts)

```typescript
// Counter: monotonically increasing number
// "How many HTTP requests have been made?" (never decreases)
const httpRequestTotal = registerCounter(
  'http_request_total',
  'Total HTTP requests',
  ['method', 'path', 'status_class']  // labels for filtering
);
httpRequestTotal.inc({ method: 'POST', path: '/api/leads', status_class: '2xx' });

// Histogram: distribution of values across buckets
// "How long did requests take? What % were under 100ms?"
const httpRequestDuration = registerHistogram(
  'http_request_duration_ms',
  'Request duration in milliseconds',
  [50, 100, 250, 500, 1000, 2500, 5000],  // bucket boundaries
  ['method', 'path']
);
httpRequestDuration.observe(432, { method: 'POST', path: '/api/leads' });

// Gauge: current value (can go up and down)
// "How many jobs are currently waiting in the queue?"
const queueDepth = registerGauge(
  'bullmq_queue_depth',
  'BullMQ queue depth',
  ['queue', 'state']
);
queueDepth.set({ queue: 'agent-research', state: 'waiting' }, 12);
```

### What's Measured

Automatically by `tracingMiddleware`:
- `http_request_total` — by method, path, status class (2xx, 4xx, 5xx)
- `http_request_duration_ms` — how long each request took

Automatically by `queueDepth.ts` (every 15s):
- `bullmq_queue_depth` — waiting + active count per queue

You can add custom metrics anywhere with `registerCounter()`, `registerGauge()`, `registerHistogram()`.

### What You Can Graph in Grafana

- Request rate: "HTTP requests per second to POST /api/leads"
- Error rate: "% of requests returning 5xx"
- Latency percentiles: "p50, p90, p99 response time"
- Queue backlog: "How many jobs waiting in agent-research queue?"
- Job failure rate: from BullMQ's own counters

---

## 2. Distributed Tracing

**File:** [`src/shared/tracing/traceContext.ts`](../../pipeclose-backend/src/shared/tracing/traceContext.ts)

A trace is a record of a single request's journey through the system — what code ran, how long each step took, what errors occurred.

### AsyncLocalStorage — How Trace Context Flows

`AsyncLocalStorage` is a Node.js API for storing data that flows automatically through an async call chain without being passed as function arguments.

```typescript
import { AsyncLocalStorage } from 'async_hooks';

const als = new AsyncLocalStorage<TraceContext>();

// Start a trace (at request begin):
als.run({ traceId: 'abc-123', companyId: 5, method: 'POST', path: '/api/leads' }, () => {
  // Everything that runs synchronously or asynchronously inside this callback
  // can call getTraceContext() and get the same context object
  next();
});

// Anywhere downstream (controller, service, model — no prop drilling needed):
function getTraceContext(): TraceContext | undefined {
  return als.getStore();
}
```

This is the "ambient" part — the trace ID is available everywhere without being explicitly passed. You don't need to add `traceId` as a parameter to every function.

### Trace Lifecycle

```
HTTP request arrives
  │
  ▼ tracingMiddleware
  Generate traceId = randomUUID()
  als.run({ traceId, ... }, () => {
    record request started
    └── next() → route handlers run
    └── on response 'finish':
          write SystemEvent { traceId, status, durationMs, statusCode }
  })
  
  │
  ▼ authMiddleware (after user identified)
  attachTraceActor({ companyId, userId, method, path })
  → writes RUNNING trace row to SystemEvent table (creates the record)
  
  │
  ▼ [controller runs, service runs, DB queries run]
  
  │
  ▼ response sent
  logTraceTerminal(traceId, { status: 'SUCCESS', durationMs: 432 })
  → updates SystemEvent row with final status and duration
```

**Why write to a separate logs database?**

SystemEvent writes go to `LOGS_DATABASE_URL`, not the main database. Reason: a burst of 1000 concurrent requests generates 1000 log writes. If those compete with `SELECT * FROM Lead` queries on the same connection pool, CRM queries slow down. The logs DB has its own pool (5 connections) so it can't starve the main pool.

### Searching Traces

In production, you can query the SystemEvent table:
```sql
SELECT * FROM SystemEvent WHERE traceId = 'abc-123';
-- See every log event from that one request

SELECT * FROM SystemEvent 
WHERE companyId = 5 AND status = 'FAILED' AND createdAt > NOW() - INTERVAL '1 hour';
-- All failures in the last hour for a company
```

---

## 3. Structured Event Logging

**File:** [`src/shared/tracing/eventLogger.ts`](../../pipeclose-backend/src/shared/tracing/eventLogger.ts)

The event logger writes structured JSON log events to the database. "Structured" means the events have typed fields, not just freeform text strings.

### Why Not Just `console.log`?

`console.log('Lead created: 42')` — this is a string. You can't query "all lead creation events for company 5 in the last hour." You'd have to grep log files.

With structured logging:
```typescript
eventLogger.log({
  type: 'LEAD_CREATED',
  companyId: 5,
  entityType: 'Lead',
  entityId: 42,
  userId: 99,
  traceId: 'abc-123',
  metadata: { name: 'Aditya', email: 'a@example.com' }
});
```

This writes to `SystemEvent`. Now you can query: "Show me all LEAD_CREATED events for company 5 from last week." The log is a queryable database, not a grep-able file.

### Buffering

Writing to the database on every log event would be slow. The event logger **buffers** events in memory:

```typescript
let buffer: SystemEvent[] = [];

function log(event: SystemEvent) {
  buffer.push(event);
  if (buffer.length >= BATCH_SIZE) {
    flush(); // write batch to DB
  }
}

// Flushed periodically and on process exit (flushLogsBuffer())
```

On graceful shutdown (`flushLogsBuffer()`), the remaining buffer is flushed before the process exits.

**Trade-off:** If the process crashes without graceful shutdown, buffered events are lost. This is acceptable — individual log events are low-value. A crash will be noticed from metrics and process monitoring. The ACID-level guarantees are only for business data (PostgreSQL main), not for logs.

---

## 4. Audit Log

**File:** [`src/shared/tracing/repository/auditLogRepository.ts`](../../pipeclose-backend/src/shared/tracing/repository/auditLogRepository.ts)

The audit log is a compliance record: **who did what, when, to which entity, and from where.**

Unlike the event log (high-volume, low-retention), the audit log is:
- Low-volume (only significant business actions)
- Long-retention (months/years, for compliance)
- Stored in the main database (not the logs DB)

```typescript
await createAudit({
  action: 'LEAD_DELETED',
  entityType: 'Lead',
  entityId: 42,
  userId: 99,
  companyId: 5,
  traceId: 'abc-123',
  before: { name: 'Aditya', status: 'OPEN' },  // snapshot before deletion
  after: null,
});
```

The `before` snapshot is important for compliance: "what was the lead's data when it was deleted?"

**Audit actions** (`src/shared/tracing/auditActions.ts`):
- `LEAD_CREATED`, `LEAD_UPDATED`, `LEAD_DELETED`
- `DEAL_CREATED`, `DEAL_WON`, `DEAL_LOST`
- `EMAIL_SENT`, `EMAIL_DELETED`
- `USER_LOGIN`, `USER_LOGOUT`, `PASSWORD_CHANGED`
- `PERMISSION_GRANTED`, `PERMISSION_REVOKED`

Some actions are **destructive** and require special handling (e.g., the before-snapshot must always be captured for legal hold purposes).

---

## 5. Winston Logger

**File:** [`src/shared/logger.ts`](../../pipeclose-backend/src/shared/logger.ts)

For immediate console/file logging (not database), the system uses [Winston](https://github.com/winstonjs/winston):

```typescript
import logger from '../shared/logger';

logger.info('User logged in', { userId: 42, companyId: 5 });
logger.warn('Redis connection error', { error: err.message });
logger.error('Unhandled exception', { stack: err.stack });
```

Winston outputs structured JSON in production (for log aggregation tools like Datadog, Splunk) and human-readable format in development.

The difference from the event logger:
- Winston → stdout/stderr (immediate, not persisted to DB, used by DevOps/log aggregation)
- Event logger → database (queryable, long-term, for business analytics and debugging)

---

## The Full Picture

```
HTTP Request
     │
     ▼ tracingMiddleware
     Generate traceId
     Start timer
     Increment http_request_total counter (Prometheus)
     
     │
     ▼ (request runs through middleware + controller + service)
     
     Each step can:
     ├── logger.info(...)         → stdout (Winston → log aggregator)
     ├── eventLogger.log(...)     → buffer → SystemEvent table (queryable)
     └── createAudit(...)         → AuditLog table (compliance)
     
     │
     ▼ Response sent
     Observe http_request_duration (Prometheus histogram)
     logTraceTerminal → UPDATE SystemEvent {status, durationMs}
     
Every 15s:
     queueDepth collector → UPDATE bullmq_queue_depth gauge (Prometheus)
     
Prometheus scrapes /metrics every 15s → Grafana graphs it
```

---

## Summary: Four Observability Tools

| Tool | Storage | Purpose | Retention |
|---|---|---|---|
| Prometheus | In-memory time series | Current state, trends, alerting | ~2 weeks of metrics |
| Event log (SystemEvent) | Logs PostgreSQL | Per-request trace, queryable history | Days/weeks (configurable) |
| Audit log (AuditLog) | Main PostgreSQL | Compliance record of significant actions | Months/years |
| Winston | stdout/log files | Developer debugging, immediate | Until log rotation |
