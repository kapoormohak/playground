# Low-Level Design (LLD) — Index

> **What is an LLD?**
> A Low-Level Design zooms into the *implementation* — the actual files, code patterns, data structures, and design decisions. Where the HLD says "the auth middleware checks your token," the LLD says exactly which function does it, what it reads from the database, what it writes to Redis, and why.

---

## Reading Order

Read the HLD files first. Then come here for the "how exactly" of each part.

| # | File | What You'll Learn |
|---|---|---|
| 1 | [01-processes.md](./01-processes.md) | server.ts, worker.ts, agent.ts — startup sequence, what each initializes, graceful shutdown |
| 2 | [02-http-layer.md](./02-http-layer.md) | Express setup, middleware chain order, route registration, controllers, services, models — the full code pattern |
| 3 | [03-database.md](./03-database.md) | Prisma client setup, connection pooling, transactions, the tenant scope guard, two DB clients |
| 4 | [04-redis-and-bullmq.md](./04-redis-and-bullmq.md) | Redis client setup, BullMQ job lifecycle, retries, deduplication via jobId |
| 5 | [05-auth.md](./05-auth.md) | JWT structure and signing, tokenVersion invalidation, session cookies, "remember me" tokens, RBAC permissions |
| 6 | [06-multi-tenancy.md](./06-multi-tenancy.md) | Three-layer tenant isolation: HTTP middleware → Prisma guard → RBAC, plus Redis caching of tenant lookups |
| 7 | [07-email-system.md](./07-email-system.md) | IMAP/SMTP protocols, 5 BullMQ queues, OAuth token lifecycle, email threading, the sync outbox pattern |
| 8 | [08-realtime.md](./08-realtime.md) | Socket.io setup, Redis adapter, cross-process event emission, WebSocket rooms and namespaces |
| 9 | [09-cron-jobs.md](./09-cron-jobs.md) | All scheduled tasks — which process runs them, what they do, failure handling |
| 10 | [10-observability.md](./10-observability.md) | Prometheus metrics, distributed tracing via AsyncLocalStorage, structured event logging, audit log |

---

## Companion HLD

| LLD Deep-Dive | ← HLD Context |
|---|---|
| [01-processes.md](./01-processes.md) | [hld/02-system-overview.md](../hld/02-system-overview.md) |
| [02-http-layer.md](./02-http-layer.md) | [hld/05-request-lifecycle.md](../hld/05-request-lifecycle.md) |
| [03-database.md](./03-database.md) | [hld/03-data-stores.md](../hld/03-data-stores.md) |
| [04-redis-and-bullmq.md](./04-redis-and-bullmq.md) | [hld/03-data-stores.md](../hld/03-data-stores.md) |
| [05-auth.md](./05-auth.md) | [hld/05-request-lifecycle.md](../hld/05-request-lifecycle.md) |
| [06-multi-tenancy.md](./06-multi-tenancy.md) | [hld/04-modules-map.md](../hld/04-modules-map.md) |
| [07-email-system.md](./07-email-system.md) | [hld/04-modules-map.md](../hld/04-modules-map.md) |
| [08-realtime.md](./08-realtime.md) | [hld/02-system-overview.md](../hld/02-system-overview.md) |
| [09-cron-jobs.md](./09-cron-jobs.md) | [hld/02-system-overview.md](../hld/02-system-overview.md) |
| [10-observability.md](./10-observability.md) | [hld/02-system-overview.md](../hld/02-system-overview.md) |
