# LLD 04 — Redis and BullMQ: Queues, Jobs, Retries, Deduplication

> **Companion HLD:** [hld/03-data-stores.md](../hld/03-data-stores.md)

---

## What Is Redis?

Redis (Remote Dictionary Server) is an in-memory key-value store. Every piece of data you store in Redis is:
- A **key** (a string)
- A **value** (a string, list, hash, set, sorted set, etc.)

"In-memory" means Redis stores everything in RAM. RAM is 100-1000x faster than disk. The trade-off: if the server reboots, data that isn't persisted to disk is lost.

Redis supports optional disk persistence (`RDB` snapshots or `AOF` append-only log), but in this system Redis is treated as ephemeral — the critical data always has a PostgreSQL copy.

---

## How Redis Is Used in This System

### 1. Caching

A cache stores the result of an expensive operation so you can skip the operation next time.

```
Request #1 (miss):
  Check Redis for key "tenant:domain:acme" → NOT FOUND
  Query PostgreSQL → { id: 5, name: "Acme Corp", ... }
  Write to Redis: SET "tenant:domain:acme" "{...}" EX 120
  
Request #2 (hit):
  Check Redis for key "tenant:domain:acme" → FOUND (instant)
  Return cached value without touching PostgreSQL
```

`EX 120` means the key expires automatically after 120 seconds. This is a **TTL (Time-To-Live)**. Without TTL, stale data would linger forever.

**What's cached:**
```
tenant:domain:{slug}     → company record, 120s TTL
company:auth:{companyId} → company record, 60s TTL
```

### 2. Socket.io Pub/Sub

Redis pub/sub is a messaging system:
- **Publisher**: emits a message on a channel
- **Subscriber**: listens on a channel and receives all messages

```
Agent Process                    Redis Pub/Sub                    Web Process
(no browser connection)           (relay)                         (has browser socket)
        │                            │                                    │
        │  PUBLISH channel msg ──────►                                    │
        │                            │── SUBSCRIBE delivers to ──────────►│
        │                            │                                    │ socket.emit(userId, msg)
        │                            │                                    │──────────────────────────► Browser
```

Socket.io's Redis adapter manages the publish/subscribe automatically. When you call `io.to(userId).emit('event', data)` in the Agent process, the adapter:
1. Serializes the event
2. Publishes it to a Redis channel
3. All Web process instances subscribed to that channel receive it
4. The one that has the user's WebSocket delivers it

### 3. BullMQ Job Queues

This is the most important use of Redis in this system. See the next section.

---

## What Is BullMQ?

BullMQ is a Node.js job queue library that uses Redis as its storage backend. A **job queue** is a list of work items (jobs) where:
- **Producers** add jobs to the queue (put work in)
- **Workers** pull jobs from the queue and execute them (take work out)

This decouples the producer from the worker. The producer doesn't need to know how the work gets done or who does it. The worker doesn't need to know who asked for the work.

```
Producer: "I need to sync Gmail for user #42"
          ──── adds job to queue ────►  [Redis Queue]
                                               │
                                               │ (when a worker is free)
                                               ▼
                                   Worker: processes the sync
```

---

## BullMQ Concepts

### Queue

A Queue is a named list in Redis. You create one per logical job type:
```typescript
const emailSyncQueue = new Queue('email:sync:historical', { connection: redis });
const researchQueue  = new Queue('agent-research',        { connection: redis });
```

Internally, Redis stores this as sorted sets (for ordering by priority/delay) and hashes (for job data).

### Job

When you add a job to a queue:
```typescript
await queue.add('process-email', {
  userId: 42,
  messageId: 'gmail:abc123',
  companyId: 5,
}, {
  jobId: 'gmail:abc123',  // custom ID for deduplication
  attempts: 3,
  backoff: { type: 'exponential', delay: 5000 },
});
```

BullMQ stores the job in Redis with:
- State: `waiting` (not yet picked up)
- Payload: `{ userId: 42, messageId: 'gmail:abc123', ... }`
- Options: max attempts, backoff strategy, etc.

### Worker

A Worker is the consumer that processes jobs:
```typescript
const worker = new Worker('email:sync:historical', async (job) => {
  // This function runs for every job pulled from the queue
  const { userId, messageId, companyId } = job.data;
  await emailService.syncMessage(userId, messageId, companyId);
  // If this function returns (doesn't throw), job is marked COMPLETED
  // If this function throws, job is marked FAILED and may be retried
}, {
  connection: redis,
  concurrency: 5, // process up to 5 jobs simultaneously
});
```

### Job Lifecycle

```
waiting → active → completed
                └► failed → (retry) → waiting → active → completed
                                   └► (max retries exceeded) → failed permanently
```

- `waiting`: in the queue, not yet picked up
- `active`: a worker is currently processing this job
- `completed`: finished successfully
- `failed`: threw an error; may be retried

---

## Retries and Backoff

**Why retries?** A job might fail transiently — network timeout, external API rate limit, database momentarily unavailable. Retrying with a delay usually succeeds.

All agent queues are configured with:
```typescript
attempts: 3,
backoff: { type: 'exponential', delay: 5_000 }
// Retry 1: wait  5,000ms (5s)
// Retry 2: wait 10,000ms (10s)
// Retry 3: wait 20,000ms (20s)
// After that: permanently failed
```

Exponential backoff prevents thundering herd problems. If 100 jobs all fail simultaneously (e.g., external API down), they don't all retry at the same second — they spread out exponentially.

---

## Deduplication via jobId

**Problem:** The AgentOutbox Relay might push the same outbox row twice. How?
1. Relay picks up row, enqueues to Redis, sets row to `DONE`
2. Crash between "enqueue to Redis" and "set to DONE"
3. On restart, row is still `PENDING`, relay picks it up again
4. Tries to enqueue the same job to Redis again

**Solution:** Every agent job is enqueued with `jobId = outboxId`:
```typescript
await queue.add('job', payload, { jobId: outboxId });
```

BullMQ's behavior: if a job with this `jobId` already exists (in any state: waiting, active, completed), the new `add()` call is **silently ignored**. The duplicate is dropped automatically.

**The `removeOnComplete: { count: 1000 }` trade-off:**

If you set `removeOnComplete: true`, completed jobs are deleted from Redis immediately. This is memory-efficient, but breaks deduplication — if the job is gone, BullMQ can't check "has this job already run?" and a redelivered outbox row would enqueue a second execution.

`{ count: 1000 }` keeps the last 1000 completed jobs visible in Redis for dedup purposes, without accumulating unbounded data.

---

## The Redis Client Setup

**File:** [`src/shared/redisClient.ts`](../../pipeclose-backend/src/shared/redisClient.ts)

```typescript
let redis: Redis | null = null;

export const getRedisClient = (): Redis => {
  if (redis) return redis;  // return existing connection (singleton)
  
  redis = new Redis(process.env.REDIS_URL || {
    host: 'localhost',
    port: 6379,
  });

  redis.on('error', (err) => {
    console.warn('Redis connection error:', err.message);
    // Non-fatal: Redis is used for caching and queues.
    // The system degrades gracefully (cache miss → DB query) if Redis is down.
  });

  return redis;
};
```

**ioredis** is the Redis client library. It maintains one persistent TCP connection to Redis. All Redis commands go through this one connection (Redis is single-threaded, so one connection is fine for most workloads).

BullMQ creates its own Redis connections (separate from the cache client) because:
- BullMQ needs `maxRetriesPerRequest: null` (don't give up on blocked commands)
- It's safer to not share a connection with cache code that might have different error handling

---

## The Queue Depth Collector

**File:** `src/shared/metrics/queueDepth.ts`

Prometheus metric: "how many jobs are currently waiting in each queue?"

```typescript
// Every 15 seconds:
for (const [queueName, queue] of registeredQueues) {
  const waiting = await queue.getWaitingCount();
  const active  = await queue.getActiveCount();
  queueDepthGauge.set({ queue: queueName, state: 'waiting' }, waiting);
  queueDepthGauge.set({ queue: queueName, state: 'active' }, active);
}
```

This is exposed via Prometheus and can be graphed in Grafana. If `agent-research` has 500 waiting jobs and only 1 active, your Research Agent worker is too slow (or crashed).

---

## When Redis Is Down

The system is designed to degrade gracefully:

**Caching:** Cache reads are wrapped in try/catch. On error, fall through to a direct PostgreSQL query. Slower, but correct.

**BullMQ:** If Redis is down, `queue.add()` will throw. The Transactional Outbox Pattern saves you here — the `agent_outbox` row is already in PostgreSQL. When Redis comes back, the relay picks up the pending rows and enqueues them. No jobs are lost.

**Socket.io pub/sub:** Real-time events stop being delivered. The browser doesn't get push notifications. Users need to manually refresh. Annoying, but not data-corrupting.

---

## Next

- [05-auth.md](./05-auth.md) — JWT, session tokens, RBAC
- [03-database.md](./03-database.md) — Prisma and the Transactional Outbox Pattern in the database
