# HLD 04 — Modules Map

> **This backend has 23 modules.** Each module owns a specific business domain — its own routes, controllers, services, and models. No module should directly touch another module's database tables.

---

## What Is a "Module"?

In this codebase, a module is a folder under `src/modules/` that contains everything related to one business domain:

```
src/modules/leads/
├── routes/        ← URL definitions (GET /leads, POST /leads/:id)
├── controllers/   ← Handle the HTTP request, call a service, return response
├── services/      ← Business logic lives here
├── models/        ← Database query wrappers (Prisma calls)
├── contracts/     ← TypeScript types for request/response shapes
└── validations/   ← Zod/Joi schemas for input validation
```

This pattern is called **separation of concerns** — each layer has one job. A route knows nothing about SQL. A model knows nothing about HTTP.

---

## The 23 Modules

### Core CRM

| Module | Path | What It Owns |
|---|---|---|
| **leads** | `modules/leads` | Leads — the central entity. Creation, update, status (open/won/lost), scoring, folders, activity history |
| **companies** | `modules/companies` | The Company model — used for two things: (1) a contact's company, (2) the tenant (Pipeclose customer) |
| **pipelines** | `modules/pipelines` | Sales pipelines, pipeline stages, deals, deal history, products, labels |
| **activities** | `modules/activities` | Activities: calls, meetings, tasks, notes — attached to leads or deals |
| **management** | `modules/management` | Contact records: Persons and Organizations |
| **prospects** | `modules/prospects` | Prospect/contact import and enrichment |

### Auth & Access

| Module | Path | What It Owns |
|---|---|---|
| **auth** | `modules/auth` | Login, logout, OTP verification, password reset, JWT issuance, session cookies, "remember me" tokens |
| **rbac** | `modules/rbac` | Role-Based Access Control — roles, permissions, per-user grants |
| **invitations** | `modules/invitations` | Team invitations — invite a colleague to join a company workspace |
| **subscriptions** | `modules/subscriptions` | Subscription tiers (free, pro, enterprise) |
| **payments** | `modules/payments` | Payment integration (Stripe or similar) |

### Communication

| Module | Path | What It Owns |
|---|---|---|
| **email** | `modules/email` | The largest module. Gmail/IMAP sync, SMTP send, drafts, signatures, email tracking (open/click pixels), scheduled emails, threading |
| **sequences** | `modules/sequences` | Email sequences — automated multi-step outreach campaigns. Steps, enrollment, scheduling |
| **calendar** | `modules/calendar` | Google Calendar two-way sync, calendar events, meeting links |
| **calls** | `modules/calls` | Call logging, call recording metadata, webhook handlers (from phone providers) |
| **notifications** | `modules/notifications` | In-app notification system (delivered via Socket.io) |

### AI Layer

| Module | Path | What It Owns |
|---|---|---|
| **agent-layer** | `modules/agent-layer` | All AI agents: research, qualification, strategy, outreach, intent, execution. Also the outbox relay, proposal engine, and execution dispatcher |
| **ai-agent** | `modules/ai-agent` | Structured knowledge base — the configuration/instructions given to AI agents |

### Platform

| Module | Path | What It Owns |
|---|---|---|
| **integrations** | `modules/integrations` | Third-party integrations (e.g., CRM imports, external data sources) |
| **import** | `modules/import` | Bulk CSV/spreadsheet import of leads |
| **export** | `modules/export` | Bulk data export |
| **observability** | `modules/observability` | Prometheus metrics endpoint, trace log viewer |

---

## Module Relationships

Modules are not fully isolated — some depend on others. But these dependencies go in one direction (a lower-level module should not import from a higher-level one).

```
                    [leads]
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
      [pipelines] [activities] [email]
          │                      │
          ▼                      ▼
      [management]           [sequences]
```

The **email** module is a special case — it's nearly a system within a system. It has its own models, services, background workers, routes, cron jobs, and BullMQ queue management.

The **agent-layer** module is also large and complex — it depends on leads, email, sequences, and pipelines to execute approved agent actions.

---

## The Shared Directory

Not everything belongs in a module. `src/shared/` contains utilities used across many modules:

```
src/shared/
├── prisma.ts           ← The Prisma client singleton (database connection)
├── redisClient.ts      ← The Redis client singleton
├── logger.ts           ← Structured logger (Winston)
├── socketInstance.ts   ← Socket.io singleton
├── killSwitches.ts     ← Feature flags stored in DB
├── agentTenantEnablement.ts ← Per-tenant AI feature flags
├── middleware/         ← Express middleware (auth, tenant, tracing, rate limit)
├── responses/          ← Standardized HTTP response helpers
├── types/              ← Shared TypeScript types
├── tracing/            ← Distributed tracing, event logging, audit
├── metrics/            ← Prometheus metric collectors
└── outbox/             ← Shared outbox poller state machine
```

---

## How a New Feature Fits In

Imagine you need to add a "Notes" feature — users can attach text notes to a lead.

Under this architecture you would:
1. Add a `Note` table to `prisma/schema.prisma`
2. Create `src/modules/notes/` with routes, controller, service, model
3. Wire the routes into `src/server.ts`
4. If notes need to trigger AI analysis, add agent_outbox writes in the service

The leads module itself doesn't need to change. That's the value of modules — features are additive, not invasive.

---

## Next

- [05-request-lifecycle.md](./05-request-lifecycle.md) — follow one HTTP request from browser to database and back
- [lld/02-http-layer.md](../lld/02-http-layer.md) — how routes, controllers, and services actually work in code
