# High-Level Design (HLD) — Index

> **What is an HLD?**
> A High-Level Design is the "bird's eye view" of a system. It answers:
> *What are the moving parts? What does each part do? How do they talk to each other?*
> It deliberately skips implementation details — that's what the LLD is for.

---

## Reading Order

Start here if you are new to this codebase. Read these files in order:

| # | File | What You'll Learn |
|---|---|---|
| 1 | [01-what-is-a-backend.md](./01-what-is-a-backend.md) | What a backend is, the client-server model, and where this backend fits |
| 2 | [02-system-overview.md](./02-system-overview.md) | The three OS processes and how they divide responsibilities |
| 3 | [03-data-stores.md](./03-data-stores.md) | PostgreSQL vs Redis — what each stores and why two databases |
| 4 | [04-modules-map.md](./04-modules-map.md) | All 23 modules, what they own, and how they relate |
| 5 | [05-request-lifecycle.md](./05-request-lifecycle.md) | The end-to-end journey of a single HTTP request |

---

## Companion LLD

Each HLD file has a matching LLD file that zooms into the implementation:

| HLD Topic | → LLD Deep-Dive |
|---|---|
| Processes | [lld/01-processes.md](../lld/01-processes.md) |
| HTTP Layer | [lld/02-http-layer.md](../lld/02-http-layer.md) |
| Database | [lld/03-database.md](../lld/03-database.md) |
| Redis & Queues | [lld/04-redis-and-bullmq.md](../lld/04-redis-and-bullmq.md) |
| Auth | [lld/05-auth.md](../lld/05-auth.md) |
| Multi-Tenancy | [lld/06-multi-tenancy.md](../lld/06-multi-tenancy.md) |
| Email System | [lld/07-email-system.md](../lld/07-email-system.md) |
| Real-time | [lld/08-realtime.md](../lld/08-realtime.md) |
| Cron Jobs | [lld/09-cron-jobs.md](../lld/09-cron-jobs.md) |
| Observability | [lld/10-observability.md](../lld/10-observability.md) |
