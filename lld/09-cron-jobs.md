# LLD 09 — Cron Jobs: All Scheduled Background Tasks

> **Companion HLD:** [hld/02-system-overview.md](../hld/02-system-overview.md)

---

## What Is a Cron Job?

A cron job is a task that runs on a schedule, not in response to an HTTP request. They handle:
- **Cleanup**: delete old data, reset daily counters
- **Maintenance**: refresh expiring tokens, renew IMAP connections
- **Recovery**: fix stuck/orphaned records that manual errors or crashes left behind
- **Proactive work**: send score decay, process upcoming scheduled items

In this codebase, cron jobs are implemented using two mechanisms:
1. **`setInterval(fn, ms)`** — Node.js built-in, runs every N milliseconds
2. **`node-cron`** — cron expression syntax (`0 0 * * *` = "at midnight daily")

---

## Web Process Crons (`src/server.ts`)

These run inside the HTTP server process. If you run 3 web instances behind a load balancer, all 3 run these crons. The jobs are designed to be idempotent (safe to run multiple times) to handle this.

---

### `tokenRefresh` (`cron/tokenRefresh.ts`) — Every 30 min

**What it does:**  
Finds Google OAuth tokens (`OAuthToken` rows) that will expire within the next 10 minutes and proactively refreshes them.

**Why?**  
Google access tokens expire in 1 hour. If you only refresh on demand, the first request after expiry is slow (must refresh, then retry). Proactive refresh keeps tokens always fresh.

```
SELECT * FROM OAuthToken WHERE expiresAt < (NOW() + 10min)
For each:
  → call Google OAuth refresh endpoint
  → UPDATE OAuthToken SET accessToken=..., expiresAt=...
```

---

### `emailSync` (`cron/emailSync.ts`) — Every 5 min

**What it does:**  
For users with IMAP connections that don't support IDLE (non-Gmail providers), this polls for new emails:
```
For each connected user:
  → connect to IMAP
  → check for new messages since last sync
  → enqueue to email:sync:realtime queue
```

For Gmail users, IDLE push handles this — polling is a fallback.

---

### `failedEmailRetry` (`cron/failedEmailRetry.ts`) — Every 10 min

**What it does:**  
Finds emails that failed to send (SMTP error, rate limit) and retries them. Checks if the account's send cap allows retrying.

---

### `gmailWatchRenewal` (`cron/gmailWatchRenewal.ts`) — Daily

**What it does:**  
IMAP IDLE connections time out after 29 minutes. Gmail also has a separate "watch" mechanism (Gmail API push notifications) that requires periodic renewal. This cron renews those watches before they expire.

---

### `trashCleanup` (`cron/trashCleanup.ts`) — Daily

**What it does:**  
Permanently deletes records that were soft-deleted more than 30 days ago. "Soft delete" means the record has a `deletedAt` timestamp but isn't actually removed from the database — it's still there but hidden from queries. Hard delete happens here.

---

### `leadScoreDecay` (`cron/leadScoreDecay.ts`) — Daily

**What it does:**  
Applies time-based decay to lead scores. A lead that hasn't had any recent activity should have its score decreased — it's "cooling off." 

```
For each company's leads where last activity > 30 days:
  newScore = currentScore * decayFactor  (e.g., 0.95 per week)
  UPDATE Lead SET score = newScore
```

This is the "time heals old scores" mechanism. Without it, leads from 2 years ago would permanently have high scores from their initial activity burst.

---

### `outboxReconciliation` (`cron/outboxReconciliation.ts`) — Every 5 min

**What it does:**  
Finds `sync_outbox` rows (the email sync outbox, not the agent outbox) that are stuck in `IN_PROGRESS` for too long — which means the relay or worker that claimed them crashed.

```
UPDATE SyncOutbox
SET status = 'PENDING'
WHERE status = 'IN_PROGRESS'
  AND updatedAt < NOW() - INTERVAL '10 minutes'
```

This is the crash recovery mechanism for the email sync outbox.

---

### `sendCapReset` (`cron/sendCapReset.ts`) — Daily at midnight

**What it does:**  
Resets email send counters. Each company has a daily limit on how many emails they can send (to prevent spam abuse). At midnight, the counter resets to 0.

---

### `reprocessUnlinkedEmails` (`cron/reprocessUnlinkedEmails.ts`) — Every 5 min

**What it does:**  
Finds emails that were synced but couldn't be matched to a lead at the time (no matching lead existed yet). Re-attempts the matching engine now that more leads may have been created.

```
SELECT * FROM Email WHERE linkedLeadId IS NULL AND createdAt > NOW() - 30 days
For each:
  → run emailLeadMatchingEngine.match(email)
  → if match found: UPDATE Email SET linkedLeadId = match.id
  → if no match: add to email:sync:reprocess BullMQ queue for later
```

---

### `activityReminderProcessor` (`cron/activityReminderProcessor.ts`) — Every min

**What it does:**  
Finds activity reminders (user set a reminder on a meeting, call, or task) that are due now and sends the notification via Socket.io.

---

### `purchasedContactExpiry` (`cron/purchasedContactExpiry.ts`) — Daily

**What it does:**  
Contacts purchased from a data provider (enrichment) have an expiry date. This cron marks them as expired so they're excluded from new campaigns.

---

### `summaryJobReaper` (`cron/summaryJobReaper.ts`) — Every 15 min

**What it does:**  
Cleans up stale AI summary jobs. If a summary generation was started but never completed (agent crashed), this marks them as failed so the UI doesn't show a forever-loading spinner.

---

### `orphanRecovery` (`cron/orphanRecovery.ts`) — Every 15 min

**What it does:**  
One of the more complex crons (12KB). Finds "orphaned" records — database rows that are in a broken state because of crashed operations:
- Emails with `status='SENDING'` for more than 10 minutes (send probably failed)
- IMAP sync sessions that started but never finished
- Deal stages with no pipeline parent (foreign key orphan)
- etc.

Each type of orphan has a specific recovery action (mark as failed, requeue, delete).

---

## Worker Process Crons (`src/worker.ts`)

These run in the dedicated Worker process — only one set, not multiplied by the number of web instances.

---

### `sequenceStepScheduler` (`cron/sequenceStepScheduler.ts`) — Every 1 min

**What it does:**  
The most important cron in the system. 46KB of code.

Sequences are automated email campaigns: "send email A now, email B in 3 days, email C after 1 week." The scheduler finds which steps are due and executes them.

```
Every minute:
  Find all SequenceEnrollment rows where:
    status = 'ACTIVE'
    AND nextStepDue <= NOW()
  
  For each enrollment:
    ├── If step type = 'email':
    │     call EmailConnectorService.sendEmail()
    │     update enrollment: nextStepDue = now + nextStep.delay
    │
    └── If step type = 'manual_task' (call, meeting):
          create Task record in DB
          write agent_outbox row {queue: 'outreach'}
          (Outreach Agent will write a brief for what to say)
```

The scheduler runs in the Worker process (not Web) because it does SMTP sends — a heavy, latency-sensitive operation that shouldn't compete with HTTP request handling.

---

### `gateCalibrationCron` (`agent-layer/cron/gateCalibrationCron.ts`) — Daily

**What it does:**  
Runs the Learning Agent's daily gate calibration. The AI qualification gate (the score threshold for a lead to pass to the strategy agent) is calibrated daily based on win/loss outcomes. This cron computes the new threshold and updates it.

---

### `proposalExpiryCron` (`agent-layer/cron/proposalExpiryCron.ts`) — Daily

**What it does:**  
AgentProposals (things the AI has suggested) can sit `PENDING` — waiting for human approval. If nobody approves or rejects within a configured window (e.g., 7 days), they expire:

```
UPDATE AgentProposal
SET status = 'EXPIRED'
WHERE status = 'PENDING'
  AND createdAt < NOW() - INTERVAL '7 days'
```

This prevents the proposal log from filling up with ancient unreviewed suggestions.

---

## Summary Table

| Cron | Process | Frequency | Purpose |
|---|---|---|---|
| tokenRefresh | Web | 30 min | Proactively refresh expiring OAuth tokens |
| emailSync | Web | 5 min | Poll IMAP for non-IDLE users |
| failedEmailRetry | Web | 10 min | Retry failed email sends |
| gmailWatchRenewal | Web | Daily | Renew Gmail IMAP IDLE connections |
| trashCleanup | Web | Daily | Hard-delete soft-deleted records after 30d |
| leadScoreDecay | Web | Daily | Apply time-based score decay to cold leads |
| outboxReconciliation | Web | 5 min | Reset stuck sync_outbox IN_PROGRESS rows |
| sendCapReset | Web | Daily midnight | Reset daily email send counters |
| reprocessUnlinkedEmails | Web | 5 min | Re-match emails to newly created leads |
| activityReminderProcessor | Web | 1 min | Fire due activity reminders via Socket.io |
| purchasedContactExpiry | Web | Daily | Mark expired enrichment contacts |
| summaryJobReaper | Web | 15 min | Clean up stale AI summary jobs |
| orphanRecovery | Web | 15 min | Fix broken records from crashed operations |
| sequenceStepScheduler | Worker | 1 min | Send due sequence emails, create tasks |
| gateCalibrationCron | Worker | Daily | Recalibrate AI qualification gate |
| proposalExpiryCron | Worker | Daily | Expire old unreviewed AI proposals |

---

## Design Pattern: Idempotency

Most of these crons must be idempotent — running them twice produces the same result as running them once. This matters because:
- Multiple web instances run the same crons simultaneously
- A cron might be restarted mid-run (server restart)

Example: `sendCapReset` uses `UPDATE ... WHERE resetAt < TODAY()` — it only resets rows that haven't been reset today. Running it 3 times doesn't triple-reset anything.

Example: `emailSync` uses `WHERE lastSyncedAt < NOW() - 5min` — only processes users who haven't been synced recently.

---

## Next

- [10-observability.md](./10-observability.md) — how the system monitors itself
- [01-processes.md](./01-processes.md) — where each cron is started in the startup sequence
