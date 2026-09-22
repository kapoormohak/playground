# HLD 03 — Data Stores: PostgreSQL and Redis

> **Key Question:** Why does this backend need two databases? What goes in each one?

---

## Two Very Different Tools

This backend uses two data stores that look similar on the surface (both store data) but solve completely different problems:

| | PostgreSQL | Redis |
|---|---|---|
| **Type** | Relational database | In-memory key-value store |
| **Storage** | Disk (persistent) | RAM (volatile by default) |
| **Speed** | Slower (disk I/O, complex queries) | Extremely fast (everything in RAM) |
| **Data model** | Tables with rows and columns, relationships via foreign keys | Keys mapping to strings, lists, hashes, sorted sets |
| **Strength** | Complex queries, joins, transactions, ACID guarantees | Speed, pub/sub, temporary data, queues |
| **In this system** | Source of truth for all business data | Job queues, caching, real-time event bus |

---

## PostgreSQL — The Source of Truth

PostgreSQL is the main database. Every piece of data that must survive a server restart lives here.

### What's stored in PostgreSQL

```
Business Data:
├── Users, Companies (tenants)
├── Leads
├── Deals + Pipelines + Stages
├── Emails (synced from Gmail/IMAP)
├── Activities (calls, meetings, notes)
├── Contacts (Persons, Organizations)
├── Sequences + Enrolled contacts
├── Calendar events
└── Subscriptions + Payments

AI / Agent Data:
├── agent_outbox          (pending AI jobs)
├── AgentProposal         (what the AI proposed)
├── ResearchItem          (evidence gathered by Research Agent)
├── AgentScoreSnapshot    (qualification scores over time)
└── AgentAudit            (audit trail of agent decisions)

Observability Data:
├── SystemEvent           (event log — separate DB)
└── AuditLog              (who did what)
```

### How the Schema Is Defined

All table definitions live in one file: [`prisma/schema.prisma`](../../pipeclose-backend/prisma/schema.prisma)

This is 150KB of table definitions. Prisma reads this file and:
1. Generates TypeScript types that match your tables
2. Creates a type-safe query client (`prisma.lead.findMany(...)`)
3. Tracks migrations (changes to table structure over time)

### What ACID Means (and Why It Matters)

PostgreSQL gives you **ACID guarantees**:

- **Atomic**: a transaction either fully succeeds or fully fails. No partial writes. If you're doing "deduct money from account A, add to account B" and it crashes halfway, the deduction is rolled back automatically.
- **Consistent**: the database always stays in a valid state. Foreign key constraints, uniqueness constraints, etc. are enforced.
- **Isolated**: two concurrent transactions don't see each other's partial writes.
- **Durable**: once committed, data survives a crash. PostgreSQL writes to disk before telling you "success."

This is why business data lives here and not in Redis.

### Two Database Connections

There are actually **two** PostgreSQL databases:

```
DATABASE_URL          ← Main CRM database (all business data)
LOGS_DATABASE_URL     ← Logs database (SystemEvent traces)
```

They're separate so a burst of log writes can't slow down your CRM queries. The logs DB connection pool is limited to 5 connections; the main DB gets up to 20. See [`src/shared/prisma.ts`](../../pipeclose-backend/src/shared/prisma.ts).

---

## Redis — Speed, Queues, and Pub/Sub

Redis is fundamentally different. It keeps everything in RAM, which makes it orders of magnitude faster than disk-based databases. But that means:
- You don't store critical data here that you can't reconstruct
- It's used for *temporary* or *derivative* data

### What Redis Is Used For in This System

#### 1. BullMQ Job Queues

BullMQ is a job queue library that uses Redis as its backend. A queue is a list where:
- **Producers** add jobs to the end
- **Workers** pick jobs from the front and process them

```
BullMQ Queues stored in Redis:
├── agent-research
├── agent-qualification
├── agent-strategy
├── agent-outreach
├── agent-execution
├── agent-intent
├── email:sync:historical
├── email:sync:realtime
├── email:sync:reprocess
├── email:sync:attachment
├── email:sync:outbound-write
├── calendar:sync
└── scheduled-emails
```

Each job in Redis is a small JSON blob: who created it, the payload, retry count, status.

#### 2. Caching

To avoid hitting the database on every request, frequently read data is cached in Redis with a TTL (time-to-live):

```
tenant:domain:{slug}    → company record, TTL: 120s
company:auth:{id}       → company record, TTL: 60s
```

When a request comes in for `acme.pipeclose.com`, the tenant middleware first checks Redis. If it's there, no database query needed. If not, it queries PostgreSQL and writes the result to Redis.

#### 3. Socket.io Pub/Sub

Socket.io uses Redis as a "message bus" between the Web, Worker, and Agent processes. When the Agent process calls `io.to(userId).emit('event', data)`:
1. Agent's Socket.io publishes the message to a Redis channel
2. All Web process instances subscribe to that channel
3. The Web process that holds the user's WebSocket connection receives it and forwards it to the browser

This is how real-time events work across processes that don't share memory.

---

## How They Work Together — A Concrete Example

**Scenario:** User clicks "Sync Email" in the browser.

```
1. Browser sends HTTP POST /api/emails/sync
                │
                ▼
2. Web Process handles request
   └─ Validates auth (checks JWT)
   └─ Writes a row to PostgreSQL: sync_outbox {status: 'PENDING'}
   └─ Returns 200 OK to browser immediately
                │
                ▼ (seconds later, sync_outbox poller wakes up)
3. Web Process reads PENDING rows from PostgreSQL sync_outbox
   └─ Enqueues a job to Redis (BullMQ queue: email:sync:historical)
   └─ Updates sync_outbox row to 'DONE'
                │
                ▼ (Worker process picks it up)
4. Worker Process
   └─ BullMQ worker pulls the job from Redis
   └─ Connects to Gmail IMAP, fetches emails
   └─ Writes emails to PostgreSQL
   └─ Emits Socket.io event via Redis pub/sub: "sync complete"
                │
                ▼
5. Web Process receives the Socket.io event via Redis
   └─ Forwards to browser's WebSocket: {event: 'emailSyncComplete'}
                │
                ▼
6. Browser shows "New emails synced!"
```

PostgreSQL holds the durable "I want to sync" record. Redis is the fast bus that moves the job to the right process. Together they make this work reliably even if any process crashes mid-way.

---

## Next

- [04-modules-map.md](./04-modules-map.md) — what each module does
- [lld/03-database.md](../lld/03-database.md) — deep dive into PostgreSQL, Prisma, connection pooling, transactions
- [lld/04-redis-and-bullmq.md](../lld/04-redis-and-bullmq.md) — deep dive into Redis, BullMQ job lifecycle, retries
