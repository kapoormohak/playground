# HLD 02 — System Overview: The Three Processes

> **Key Insight:** This backend is not a single program. It is three separate OS processes that share the same database and Redis instance.

---

## Why Three Separate Processes?

A naive backend design is one program that does everything — handles HTTP requests, runs background jobs, sends emails. That works at small scale. But it breaks down:

- **Scaling problem**: if email sync is slow, it hogs the same CPU/memory your HTTP server uses. Users experience slow API responses. 
- **Isolation problem**: a bug in the AI agent shouldn't crash your HTTP server.
- **Deployment problem**: you can't restart the AI agent independently without also bouncing your HTTP server.

The solution: **separate processes for separate concerns**. Each can be scaled, restarted, and deployed independently.

---

## The Three Processes

### Process 1: Web (`src/server.ts`)

**Started with:** `node dist/server.js`  
**Role:** Handles all HTTP requests from the browser. The only process the internet talks to directly.

**What it does:**
- Creates an Express HTTP server on a port (default 3000)
- Mounts all REST API routes (`/api/leads`, `/api/deals`, `/api/emails`, etc.)
- Runs lightweight cron jobs (token refresh, email sync polling, lead score decay)
- Connects to Socket.io for real-time browser events

**What it does NOT do:**
- Does not run AI agents
- Does not do heavy background processing (email sync is triggered here but processed in the Worker)

**Scale:** You can run multiple web processes behind a load balancer (nginx/AWS ALB). Each instance is stateless — any request can go to any instance.

---

### Process 2: Worker (`src/worker.ts`)

**Started with:** `NODE_ROLE=worker node dist/worker.js`  
**Role:** Consumes jobs from BullMQ queues. Does the actual heavy lifting of email sync, calendar sync, sending scheduled emails.

**What it does:**
- Initializes the EmailService singleton (required by all job processors)
- Starts BullMQ workers that listen on Redis queues:
  - Email sync (historical, realtime, reprocess, attachment, outbound-write queues)
  - Calendar sync
  - Scheduled email sending
- Runs the Sequence Step Scheduler (finds which emails to send today)
- Runs daily crons (gate calibration, proposal expiry)

**What it does NOT do:**
- Does not start an HTTP server (no web traffic comes here directly)
- Does not run AI agents

**Scale:** You can run multiple Worker processes. Each one competes for jobs on the same BullMQ queues — BullMQ is designed for this. More workers = faster throughput.

---

### Process 3: Agent (`src/agent.ts`)

**Started with:** `NODE_ROLE=agent node dist/agent.js`  
**Role:** Runs all AI pipelines. Isolated from the web server on purpose.

**What it does:**
- Sets `NODE_ROLE=agent` **before** loading any code — this activates extra database guardrails (tenant scope enforcement)
- Registers 6 BullMQ queues for AI agents
- Starts the AgentOutbox Relay (polls PostgreSQL every 3 seconds for pending AI jobs)
- Starts 6 BullMQ consumers (one per AI agent):
  - Research Agent
  - Qualification Agent
  - Campaign Strategy Agent
  - Outreach Agent
  - Execution Consumer
  - Intent Reply Agent
- Exposes Prometheus metrics on port 4002

**Why isolated?**
- AI API calls (to Perplexity, OpenAI, etc.) can be slow and expensive. They must not block your HTTP server's thread pool.
- Tenant-scope enforcement: the Agent process has stricter database guardrails that the web process intentionally doesn't have.

---

## How They Share State

All three processes talk to the **same** PostgreSQL and Redis:

```
┌────────────────┐
│  Web Process   │─────────────────────┐
│  (server.ts)   │                     │
└────────────────┘                     ▼
                                ┌─────────────┐     ┌─────────────────┐
┌────────────────┐              │ PostgreSQL  │     │     Redis        │
│ Worker Process │◄────────────►│             │◄───►│                  │
│  (worker.ts)   │              │ (all data)  │     │ (queues, cache,  │
└────────────────┘              └─────────────┘     │  Socket.io pub)  │
                                                    └─────────────────┘
┌────────────────┐                     ▲
│ Agent Process  │─────────────────────┘
│  (agent.ts)    │
└────────────────┘
```

**PostgreSQL** is the source of truth for all business data. Any process can read/write it.

**Redis** serves two purposes:
1. **Queue backend**: BullMQ uses Redis to store job queues. Worker and Agent both read from it.
2. **Pub/Sub**: Socket.io uses Redis to relay WebSocket events across processes. The Agent emits an event → Redis pub/sub → Web process receives it → sends to browser's WebSocket.

---

## The Real-time Bridge

This is subtle and worth understanding. The browser connects via WebSocket to the **Web** process (it's the only one with an HTTP listener). But the **Worker** or **Agent** might be the one doing the work (e.g., email sync finishes, agent analysis completes). How does the result reach the browser?

```
Agent Process                 Redis                  Web Process         Browser
     │                          │                        │                  │
     │  io.to(userId).emit()    │                        │                  │
     │ ─────────────────────► pub                        │                  │
     │                          │                        │                  │
     │                         sub ──────────────────────►                  │
     │                          │            socket.emit(userId, data)      │
     │                          │                        │─────────────────►│
     │                          │                        │                  │
```

All three processes create a Socket.io server, but only the Web process has a real HTTP listener. The others connect via a **Redis adapter** — they can emit events, and the Redis pub/sub system delivers those events to whichever Web process instance holds the client's actual WebSocket connection.

---

## Visual Summary

```
internet
   │
   ▼
[Load Balancer / nginx]
   │
   ├──► [Web Process #1]  ┐
   ├──► [Web Process #2]  ├── all share same PostgreSQL + Redis
   └──► [Web Process #3]  ┘
   
[Worker Process #1]  ┐
[Worker Process #2]  ├── drain BullMQ queues from Redis
[Worker Process #3]  ┘

[Agent Process #1]   ─── single instance (for now)
```

---

## Next Steps

- To understand data storage: [03-data-stores.md](./03-data-stores.md)
- To see the full module map: [04-modules-map.md](./04-modules-map.md)
- For deep implementation details of each process: [lld/01-processes.md](../lld/01-processes.md)
