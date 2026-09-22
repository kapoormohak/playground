# HLD 01 — What Is a Backend?

> **Before we talk about THIS backend, we need to understand what any backend is.**

---

## The Client-Server Model

When you open the Pipeclose web app in your browser, you are looking at a **client**. The client is the part of the software that runs *on the user's machine* — it draws the UI, handles clicks, and shows data on screen.

But the client is dumb by itself. It doesn't store your leads, it can't send emails, it doesn't know anything about other users. For all of that, it needs to talk to a **server**.

```
┌─────────────────────┐              ┌──────────────────────────┐
│   Browser (Client)  │  ─── HTTP ──►│    Backend (Server)       │
│                     │              │                           │
│  Shows the UI       │◄─── JSON ───│  Stores data in a DB      │
│  Handles clicks     │              │  Sends emails             │
│  Renders React app  │              │  Runs business logic      │
└─────────────────────┘              │  Controls who sees what   │
                                     └──────────────────────────┘
```

The client sends **HTTP requests** ("give me this lead's data", "save this new deal"). The backend does the work and sends back **JSON responses** ("here is the lead", "saved successfully").

---

## What HTTP Really Means

HTTP (HyperText Transfer Protocol) is just a text-based convention for how computers ask each other for things. Every request has:

- A **method** — what kind of operation: `GET` (read), `POST` (create), `PUT`/`PATCH` (update), `DELETE` (remove)
- A **path** — which resource: `/api/leads/123`, `/api/deals`
- **Headers** — metadata: who you are (your auth token), what language you speak, etc.
- A **body** — data you're sending (for POST/PATCH): the new lead's name, email, etc.

Every response has:
- A **status code** — `200` (OK), `201` (Created), `400` (bad request), `401` (not logged in), `403` (not allowed), `404` (not found), `500` (server crashed)
- A **body** — the JSON data or error message

---

## What This Backend Does

Pipeclose is a **CRM (Customer Relationship Management)** platform. The backend is responsible for:

| Responsibility | What It Means |
|---|---|
| **Data persistence** | Storing leads, deals, emails, companies — all the CRM data |
| **Authentication** | Knowing who you are (login, sessions, tokens) |
| **Authorization** | Knowing what you are allowed to do (admin vs. regular user) |
| **Business logic** | Rules: a lead must have an email, a deal needs a pipeline stage, etc. |
| **Email integration** | Syncing emails from Gmail/IMAP, sending emails via SMTP |
| **Calendar integration** | Two-way sync with Google Calendar |
| **AI agent orchestration** | Running AI processes on leads (research, qualification, outreach) |
| **Real-time updates** | Pushing live notifications to the browser (email arrived, lead updated) |
| **Multi-tenancy** | Keeping Company A's data 100% isolated from Company B's |

---

## Where This Backend Lives in the Repo

```
pipeclose/
├── pipeclose-frontend/   ← React app (the client, runs in your browser)
└── pipeclose-backend/    ← THIS. Runs on a server. Never in the browser.
    ├── src/
    │   ├── server.ts     ← Entry point #1: HTTP server
    │   ├── worker.ts     ← Entry point #2: background job processor
    │   ├── agent.ts      ← Entry point #3: AI agent runner
    │   ├── modules/      ← Business features (leads, email, auth, etc.)
    │   ├── shared/       ← Utilities used everywhere (DB client, middleware, etc.)
    │   ├── cron/         ← Scheduled jobs (run every X minutes)
    │   └── infrastructure/ ← Cross-cutting concerns (event bus, encryption)
    └── prisma/
        └── schema.prisma ← The database table definitions
```

---

## The Technology Stack

| Layer | Technology | What It Is |
|---|---|---|
| Language | TypeScript | JavaScript with type-checking. Compiles to Node.js. |
| Runtime | Node.js | JavaScript runtime — runs server-side JS code |
| HTTP Framework | Express.js | The most popular Node.js web framework. Handles routing, middleware, request/response. |
| Database | PostgreSQL | The main relational database. All CRM data lives here. |
| ORM | Prisma | A type-safe query builder that talks to PostgreSQL. No raw SQL needed. |
| Cache / Queue | Redis | An in-memory data store. Used as a cache AND as a job queue backend. |
| Job Queue | BullMQ | A job queue library backed by Redis. For running work in the background. |
| Real-time | Socket.io | WebSocket library for pushing live events to the browser. |
| Email Protocol | IMAP / SMTP | IMAP = read emails. SMTP = send emails. Standard protocols, not API-based. |

---

## Next: The Three Processes

The most unusual thing about this backend is that it runs as **three separate programs** simultaneously. Learn about that in [02-system-overview.md](./02-system-overview.md).
