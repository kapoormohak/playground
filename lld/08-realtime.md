# LLD 08 — Real-time: Socket.io and the Redis Adapter

> **Companion HLD:** [hld/02-system-overview.md](../hld/02-system-overview.md)

---

## What Problem Does Real-time Solve?

HTTP is a request-response protocol. The client asks, the server answers. The server cannot proactively push data to the client.

But real-time features require the server to push:
- "You just received a new email"
- "Someone else updated this lead"
- "Your email sync just finished"
- "The AI agent completed analysis on this lead"

Without real-time, the only option is **polling** — the frontend calls `GET /api/notifications` every 5 seconds. This is wasteful (99% of polls return nothing new) and slow (you might miss an event for up to 5 seconds).

The better approach: **WebSockets** — a persistent bidirectional connection between the browser and the server. Once established, either side can send a message at any time.

---

## Socket.io

**[Socket.io](https://socket.io/)** is a library built on WebSockets. It adds:
- **Auto-reconnect**: if the connection drops, Socket.io reconnects automatically
- **Rooms**: group connections; emit to "everyone in room X" instead of listing individual connections
- **Namespaces**: logical separation within one connection
- **Fallback**: if WebSockets aren't available, falls back to HTTP long-polling

### The Core Abstraction

```typescript
// Server side: emit an event to a specific user
io.to(`user:${userId}`).emit('emailSynced', { count: 42 });

// Client side (React): listen for that event
socket.on('emailSynced', ({ count }) => {
  setEmailCount(count);
});
```

`io.to('room-name')` — "room" is just a string label. You can put any connected socket into any room. Emitting to a room sends to all sockets in that room.

---

## The Cross-Process Problem

Here's the challenge specific to this backend. There are **three OS processes**:

- **Web process**: holds the actual WebSocket connections from browsers
- **Worker process**: does the work (email sync) but has no browser connections
- **Agent process**: does AI analysis but has no browser connections

When the Worker finishes syncing emails, it needs to tell the browser. But the browser's WebSocket is connected to the Web process, not the Worker.

```
Wrong assumption:
Worker.io.to(userId).emit(...)
  → finds no socket (no browser connected to Worker)
  → event is lost
```

The solution: **Redis adapter**.

---

## The Redis Adapter

**Library:** `@socket.io/redis-adapter`

All three processes create a Socket.io instance. But instead of emitting directly to sockets, they go through Redis pub/sub:

```
Worker Process                    Redis                    Web Process
(no browser sockets)                                      (has browser sockets)
        │                            │                           │
        │  io.to(userId).emit(event) │                           │
        │                            │                           │
  Socket.io (Redis adapter)          │                           │
  serializes event                   │                           │
  PUBLISH to Redis channel ─────────►│                           │
                                     │                           │
                                     │ Socket.io (Redis adapter) │
                                     │ SUBSCRIBE delivery ───────►│
                                     │                     finds the socket for userId
                                     │                     actual WebSocket delivery ──► Browser
```

The setup in each process:

```typescript
// In worker.ts and agent.ts:
const pubClient = new Redis(REDIS_URL);
const subClient = pubClient.duplicate(); // separate connection for subscribing

const io = new SocketIOServer({ transports: ['websocket'] });
io.adapter(createAdapter(pubClient, subClient)); // attach Redis adapter

// NO http.listen() call here — this Socket.io can only emit, not receive connections
setIO(io); // register as singleton for use throughout the process
```

```typescript
// In server.ts:
const httpServer = createServer(expressApp);
const io = new SocketIOServer(httpServer, { /* cors, transports */ });
io.adapter(createAdapter(pubClient, subClient)); // same adapter setup
httpServer.listen(PORT);

// This one actually accepts WebSocket connections from browsers
```

**One critical detail:** Two separate Redis connections (`pubClient` and `subClient`). Redis pub/sub requires separate connections for publishing and subscribing — a single connection cannot do both simultaneously.

---

## Socket.io Rooms

When a browser connects, it's placed into a room named after the user:

```typescript
// In the Socket.io connection handler (server.ts):
io.on('connection', (socket) => {
  // After authenticating the socket connection:
  socket.join(`user:${userId}`);
  socket.join(`company:${companyId}`);
});
```

Two room types:
- **User room** (`user:42`): events for a specific user (your email synced)
- **Company room** (`company:5`): events for anyone in a company (team notification, new lead assigned)

Emitting to a room:
```typescript
// From anywhere in the system:
import { getIO } from '../shared/socketInstance';

const io = getIO();
io.to(`user:${userId}`).emit('emailSynced', { count: 42 });
io.to(`company:${companyId}`).emit('leadCreated', { leadId: 99, name: 'Aditya' });
```

---

## The Socket.io Singleton

**File:** [`src/shared/socketInstance.ts`](../../pipeclose-backend/src/shared/socketInstance.ts)

```typescript
let _io: SocketIOServer | null = null;

export function setIO(io: SocketIOServer): void {
  _io = io;
}

export function getIO(): SocketIOServer {
  if (!_io) throw new Error('Socket.io not initialized');
  return _io;
}
```

This singleton is set once at process startup (by `server.ts`, `worker.ts`, or `agent.ts`) and then retrieved anywhere that needs to emit events.

---

## RealTimeNotificationService

**File:** `src/modules/email/services/realTimeNotificationService.ts`

A wrapper service around the Socket.io singleton, providing named methods for each event type:

```typescript
class RealTimeNotificationService {
  private io: SocketIOServer | null = null;
  
  initialize(io: SocketIOServer) {
    this.io = io;
  }
  
  notifyEmailSynced(userId: number, count: number) {
    this.io?.to(`user:${userId}`).emit('emailSynced', { count });
  }
  
  notifyNewEmail(companyId: number, email: EmailSummary) {
    this.io?.to(`company:${companyId}`).emit('newEmail', email);
  }
}
```

The `?.` optional chaining: if `this.io` is null (Redis not configured, or called before `initialize()`), the emit is silently skipped. Real-time is a "nice to have" — its absence shouldn't crash the server.

---

## Gmail IDLE — Push vs. Poll

Without IDLE, you'd poll Gmail every 5 minutes. With IDLE, Gmail pushes a notification when a new email arrives.

**IDLE Flow:**
```
1. Email sync worker connects to IMAP with IDLE mode enabled
2. Worker sends IMAP IDLE command to Gmail
3. Gmail holds the connection open (no data exchanged)
4. New email arrives in user's inbox
5. Gmail sends a push notification on the idle connection:
   "* 1 EXISTS" (there's a new message)
6. Worker terminates IDLE mode
7. Worker fetches the new message
8. Adds job to email:sync:realtime BullMQ queue
9. Re-enters IDLE mode
```

**`gmailWatchRenewal` cron:** IMAP IDLE connections time out after 29 minutes (per RFC). The `cron/gmailWatchRenewal.ts` job renews them daily, keeping push notifications active.

---

## What Happens When Redis Is Down

The Redis adapter handles this gracefully:
- Existing WebSocket connections continue to work (the socket is still there in the Web process)
- Cross-process emission stops: Worker/Agent emit calls go into Redis and... never deliver (Redis is down)
- Web process emissions to its own connections still work (no Redis needed for same-process sockets)
- When Redis recovers, cross-process emission resumes automatically (ioredis reconnects)

---

## Next

- [09-cron-jobs.md](./09-cron-jobs.md) — all scheduled background tasks
- [07-email-system.md](./07-email-system.md) — how Socket.io events are used in email sync
