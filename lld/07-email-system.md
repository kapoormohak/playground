# LLD 07 — The Email System

> **Companion HLD:** [hld/04-modules-map.md](../hld/04-modules-map.md)  
> **Module:** `src/modules/email/`

---

## Overview

The email system is the most complex module in the backend. It handles:
- **Receiving** emails: syncing from Gmail/IMAP into the database
- **Sending** emails: SMTP outbound, with tracking
- **Scheduling** emails: queue and send at a future time
- **Sequencing** emails: automated multi-step campaigns
- **Threading**: grouping emails into conversation threads
- **Linking**: matching emails to lead/deal records

It has its own BullMQ queues, its own cron jobs, and its own OAuth token lifecycle.

---

## IMAP and SMTP — What They Are

Before the code, you need to understand the protocols.

**IMAP (Internet Message Access Protocol):**
- For *reading* emails from a mail server
- You connect to Gmail's IMAP server (`imap.gmail.com:993`) and request mailbox contents
- Supports **IDLE** — a push mechanism where Gmail notifies you when a new email arrives, instead of you polling every 5 minutes

**SMTP (Simple Mail Transfer Protocol):**
- For *sending* emails
- You connect to an SMTP server (`smtp.gmail.com:587`) and submit a message for delivery
- The SMTP server then routes it to the recipient's mail server

**OAuth (Google OAuth2):**
- Gmail now requires OAuth tokens instead of passwords for IMAP/SMTP access
- Users grant Pipeclose permission via Google's OAuth flow ("Allow Pipeclose to read your Gmail")
- Pipeclose receives an `access_token` (short-lived, 1 hour) and a `refresh_token` (long-lived)
- When the access token expires, the `OAuthService` uses the refresh token to get a new one automatically

---

## The Email Service Architecture

```
src/modules/email/services/
├── emailService.ts            ← Orchestrator (singleton)
├── oauthService.ts            ← Manages Google OAuth tokens
├── emailConnectorService.ts   ← IMAP/SMTP protocol layer
├── emailSyncQueueService.ts   ← BullMQ queue management
└── realTimeNotificationService.ts ← Socket.io events
```

### EmailService (Singleton)

`EmailService` is the orchestrator. It's instantiated once by `worker.ts` and registered as a singleton:

```typescript
// Inside EmailService constructor:
setEmailService(this); // register as the singleton

// Anywhere in the codebase:
const emailService = getEmailService(); // retrieve the singleton
```

**Why a singleton?** BullMQ workers are created independently, but they all need to call `emailService.syncMailbox()`. The singleton pattern lets each worker call `getEmailService()` without needing to construct the full service graph itself.

### OAuthService

Manages the lifecycle of Google OAuth tokens:

```typescript
class OAuthService {
  // Get a valid access token (refreshes if expired)
  async getValidAccessToken(userId: number, companyId: number): Promise<string> {
    const tokens = await prisma.oAuthToken.findFirst({
      where: { userId, companyId }
    });
    
    if (isExpired(tokens.expiresAt)) {
      const newTokens = await refreshGoogleToken(tokens.refreshToken);
      await prisma.oAuthToken.update({
        where: { id: tokens.id },
        data: {
          accessToken: newTokens.access_token,
          expiresAt: new Date(Date.now() + newTokens.expires_in * 1000)
        }
      });
      return newTokens.access_token;
    }
    
    return tokens.accessToken;
  }
}
```

The `cron/tokenRefresh.ts` job runs every 30 minutes as a proactive sweep — it finds tokens expiring soon and refreshes them before they're needed. This prevents the first request after an expiry from being slow.

### EmailConnectorService

The low-level IMAP/SMTP adapter. Creates connections using the `imap` and `nodemailer` Node.js libraries:

```typescript
class EmailConnectorService {
  async createImapConnection(userId: number, companyId: number) {
    const accessToken = await this.oauthService.getValidAccessToken(userId, companyId);
    
    return new Imap({
      host: 'imap.gmail.com',
      port: 993,
      tls: true,
      xoauth2: buildXOAuth2Token(email, accessToken) // Gmail OAuth IMAP auth
    });
  }
  
  async sendEmail(smtpConfig: SmtpConfig, message: EmailMessage) {
    const transporter = nodemailer.createTransport({ ...smtpConfig });
    return transporter.sendMail(message);
  }
}
```

---

## The Five BullMQ Queues

Email sync is split into 5 queues for different phases of work:

| Queue | Purpose | When Jobs Are Added |
|---|---|---|
| `email:sync:historical` | First-time sync: pull all old emails | User connects their Gmail |
| `email:sync:realtime` | Process new emails as they arrive | Gmail IDLE push notification |
| `email:sync:reprocess` | Retry emails that failed to link to leads | `reprocessUnlinkedEmails` cron |
| `email:sync:attachment` | Download email attachments | After email body is synced |
| `email:sync:outbound-write` | Persist sent emails back to DB | After SMTP send |

**Why separate queues?** Priority isolation. Real-time emails (new message just arrived) should be processed immediately, not stuck behind 10,000 historical sync jobs. By separating them, you can give the realtime queue higher priority or more workers.

---

## Email Sync Flow (Historical)

When a user connects their Gmail account for the first time:

```
1. User completes Google OAuth flow
   ├── Frontend redirects to Google consent screen
   ├── Google redirects back to /api/integrations/google/callback
   └── Backend stores access_token + refresh_token in OAuthToken table
   
2. Backend triggers historical sync:
   ├── Writes a row to sync_outbox table {userId, type: 'HISTORICAL', status: 'PENDING'}
   └── Returns 200 OK to the frontend immediately
   
3. syncOutboxProcessor (cron, every 5 min in web process):
   ├── Reads PENDING rows from sync_outbox
   ├── Adds a job to BullMQ queue: email:sync:historical {userId, companyId}
   └── Marks sync_outbox row as DONE
   
4. Worker process (email:sync:historical BullMQ worker):
   ├── Gets a valid OAuth access token
   ├── Connects to IMAP server
   ├── Fetches emails in batches (e.g., 50 at a time)
   ├── For each email:
   │     ├── Writes to Email table in PostgreSQL
   │     ├── Tries to match to a Lead (by email address)
   │     └── If attachment: adds job to email:sync:attachment queue
   └── Emits Socket.io event: "historicalSyncProgress" (with count)
   
5. Browser receives the Socket.io event → updates progress bar
```

The **sync_outbox** here is the same Transactional Outbox Pattern as the agent_outbox — write to the DB first (durable), then a poller reads it and pushes to BullMQ.

---

## Email Linking to Leads

When an email is synced, the system tries to match it to an existing lead record. This is non-trivial:

**File:** `src/modules/leads/services/emailLeadMatchingEngine.ts` (12KB)

Matching strategy:
1. **Exact email address match**: the sender/recipient email exactly matches a lead's email
2. **Domain match + contact match**: email from `john@bigcorp.com` where `bigcorp.com` is a known company domain
3. **Thread ID match**: if a previous email in the same thread was already linked, link this one too

If no match is found, the email goes into the "reprocess" queue to be retried later (after new leads might have been created).

---

## Email Sending (Outbound)

When a user sends an email from Pipeclose:

```
1. User clicks Send in the frontend
   └── POST /api/emails/send { to, subject, body, leadId }
   
2. Controller → EmailService.sendEmail()
   ├── Validates: user has SMTP credentials configured
   ├── Creates a draft Email row in DB (status: 'SENDING')
   └── Calls EmailConnectorService.sendEmail() via SMTP
   
3. SMTP delivers to recipient's mail server
   
4. After send: adds job to email:sync:outbound-write queue
   └── This job writes the sent email back to PostgreSQL with final status: 'SENT'
   └── Updates the lead's activity timeline
   
5. Inserts open-tracking pixel in HTML body:
   <img src="/api/emails/track/open/{emailId}" width=1 height=1>
   When recipient opens the email, their browser loads this pixel →
   backend records the open event
```

**Why the outbound-write queue?** Writing to the database after SMTP send is non-critical (the email was sent — it can't be unsent). Putting it on a queue means the HTTP response to the user is immediate, without waiting for the DB write.

---

## Scheduled Emails

**File:** `src/modules/email/workers/ScheduledEmailWorker.ts`

Users can write an email now but schedule it to send later.

```
1. User creates a scheduled email
   └── POST /api/emails/scheduled { to, subject, body, sendAt: "2025-12-01T09:00:00Z" }
   └── Backend writes: ScheduledEmail { ... status: 'PENDING', sendAt }
   
2. ScheduledEmailWorker (BullMQ worker, runs in Worker process):
   ├── BullMQ job has a `delay` set to (sendAt - now) milliseconds
   └── BullMQ doesn't process the job until the delay expires
   
3. At sendAt time:
   └── Worker processes the job → EmailConnectorService.sendEmail()
```

BullMQ's `delay` feature uses Redis sorted sets — the job is stored with a score equal to `processAfter` timestamp. Workers check if jobs are ready by comparing score to current time.

---

## Email Threading

Emails are grouped into conversation threads. Two emails are in the same thread if they share a `threadId` (Gmail's threading) or if one is a reply to the other (`In-Reply-To` header matches the first email's `Message-ID`).

This is stored in the Email model:
```prisma
model Email {
  id         Int
  threadId   String?   // Gmail thread ID
  messageId  String    // RFC 2822 Message-ID header
  inReplyTo  String?   // RFC 2822 In-Reply-To header
  // ...
}
```

The UI groups emails by `threadId` to show conversations.

---

## The Sync Outbox vs. Agent Outbox

Both follow the same pattern (Transactional Outbox → Poller → BullMQ), but:

| | `sync_outbox` | `agent_outbox` |
|---|---|---|
| **Table** | `SyncOutbox` | `AgentOutbox` |
| **Poller** | `syncOutboxProcessor.ts` (cron in web process) | `agentOutboxRelay.ts` (Agent process) |
| **Queues** | email:sync:* queues | agent-* queues |
| **Concurrency** | Cron-based, single runner | FOR UPDATE SKIP LOCKED, multiple relays possible |
| **Purpose** | Email sync jobs | AI agent jobs |

---

## Next

- [08-realtime.md](./08-realtime.md) — how Socket.io events cross process boundaries
- [09-cron-jobs.md](./09-cron-jobs.md) — the cron jobs that support the email system
