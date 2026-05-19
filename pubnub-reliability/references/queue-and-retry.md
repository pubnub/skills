<!-- canonical-for: QUEUE_AND_RETRY -->
<!-- used-by: pubnub-app-developer, pubnub-presence -->

# Queue and Retry: Offline Publish Buffer

The canonical reference for buffering publishes when offline and draining them on reconnect, without losing or duplicating.

## When You Need It

- Mobile app where the user can go through a tunnel mid-action
- Field-service / IoT publisher with intermittent connectivity
- A web app where a transient network error mid-publish would otherwise drop the user's message

If publish failures are tolerable (e.g., a live ticker where stale data isn't valuable), you don't need this. For anything user-typed or critical telemetry, you do.

## The Pattern

```
┌────────────┐                        ┌────────────────┐
│ App emits  │ → enqueue to local → │  Persistent    │
│   send()   │     storage          │  Queue         │
└────────────┘                        └────────────────┘
                                             │
                            ┌────────────────┘
                            ▼
                   ┌──────────────┐         ┌──────────────┐
                   │ Drain Worker │ → ack → │ Mark dequeued│
                   │ (online?)    │         │ (delete row) │
                   └──────────────┘         └──────────────┘
                            │
                       (offline / err)
                            ↓
                  retry with backoff+jitter
```

## Required Properties

| Property | How to satisfy |
|---|---|
| **Persistent** | localStorage / IndexedDB (web), SQLite (mobile) — never in-memory only |
| **Ordered** | FIFO drain in enqueue order |
| **Idempotent** | Each queued item carries a stable `message_id` from creation time |
| **Bounded** | Cap by count + age; surface to user when at the cap |
| **Observable** | Log queue depth periodically and on every drain attempt |

## Minimal IndexedDB Implementation

```javascript
async function enqueue(channel, message) {
  const item = {
    id: crypto.randomUUID(),       // queue-row UUID (separate from PubNub's [userId/UUID](../../pubnub-app-developer/references/sdk-patterns.md) and from message_id)
    enqueued_at: Date.now(),
    channel,
    message,                       // includes its own message_id from createMessage()
    attempts: 0
  };
  await db.put('outbox', item);
}

async function drainOnce() {
  if (!isOnline()) return;
  const pending = await db.getAll('outbox');     // FIFO by enqueue order

  for (const item of pending) {
    try {
      await pubnub.publish({ channel: item.channel, message: item.message });
      await db.delete('outbox', item.id);
    } catch (err) {
      item.attempts++;
      if (item.attempts > MAX_ATTEMPTS) {
        await db.delete('outbox', item.id);
        await db.put('dead-letter', item);
        log('queue.dead', item);
        continue;
      }
      await db.put('outbox', item);
      await sleepWithJitter(item.attempts);     // see backoff-and-jitter.md
    }
  }
}
```

## Drain Triggers

For underlying [`addListener` and `pubnub.publish`](../../pubnub-app-developer/references/publish-subscribe.md) mechanics see the canonical owner.

Drain on:

- App start
- Reconnect (`PNReconnectedCategory`) — see [pubnub-presence/references/dropped-connections.md](../../pubnub-presence/references/dropped-connections.md)
- A periodic interval (every 30s) as a backstop
- Each `enqueue` (try to drain immediately if online)

```javascript
pubnub.addListener({
  status: (s) => {
    if (s.category === 'PNReconnectedCategory' || s.category === 'PNConnectedCategory') {
      drainOnce();
    }
  }
});

setInterval(drainOnce, 30_000);
```

## Bounded Queue

If the user is offline for a long time, the queue can grow without bound. Cap it:

```javascript
const MAX_QUEUE_DEPTH = 1000;
const MAX_QUEUE_AGE_MS = 7 * 24 * 60 * 60 * 1000;   // 7 days

async function enqueue(channel, message) {
  const depth = await db.count('outbox');
  if (depth >= MAX_QUEUE_DEPTH) {
    showUserQueueFullError();
    return false;
  }
  // ...prune aged-out entries...
  return true;
}
```

When at the cap, refuse the enqueue and tell the user. Silently dropping is worse than a clear error.

## Dead-Letter Queue

Items that exhaust `MAX_ATTEMPTS` move to a dead-letter store. Don't auto-discard — surface them so a developer or the user can decide:

```javascript
async function inspectDeadLetter() {
  const dead = await db.getAll('dead-letter');
  return dead.map(item => ({
    enqueued_at: new Date(item.enqueued_at),
    age_minutes: (Date.now() - item.enqueued_at) / 60_000,
    attempts: item.attempts,
    channel: item.channel,
    message_id: item.message.message_id
  }));
}
```

## Drain Concurrency

Single-worker FIFO drain by default. Parallel drain is tempting but breaks order. If you do parallel drain (e.g., per-channel concurrency), ensure dedup at the receiver — see [dedup-on-merge.md](dedup-on-merge.md).

## Idempotency Is Mandatory

The queue can publish the same item twice if it crashes mid-publish (the publish succeeded, the queue write failed). The receiver must dedup by `message_id` — see [idempotent-publish.md](idempotent-publish.md). This is non-optional.

## Observability

Log queue depth on every drain and on a 60s interval:

```javascript
async function logQueueDepth() {
  const depth = await db.count('outbox');
  const dead  = await db.count('dead-letter');
  const oldest = depth > 0 ? await db.getOldest('outbox') : null;
  log('queue.depth', {
    outbox: depth,
    dead_letter: dead,
    oldest_age_ms: oldest ? Date.now() - oldest.enqueued_at : 0
  });
}
setInterval(logQueueDepth, 60_000);
```

For correlation field conventions and incident triage that uses queue depth, see [pubnub-observability/references/logging-correlation.md](../../pubnub-observability/references/logging-correlation.md) and [pubnub-observability/references/incident-runbook.md](../../pubnub-observability/references/incident-runbook.md).

## Anti-Patterns

| Anti-pattern | Fix |
|---|---|
| In-memory queue | Lost on app crash; persist |
| Unbounded queue | Eventually fills storage; cap by count + age |
| Auto-discarding dead letters | Move to a dead-letter store and surface |
| Parallel drain without receiver dedup | Order breaks; rely on dedup with message_id |
| No retry attempt counter | Stuck items retry forever |
| Forgetting `message_id` | Queue retries are not safe |
| Treating "publish ok" before the network ack | Mark ack only after the SDK promise resolves |

## Related Reading

- [idempotent-publish.md](idempotent-publish.md) — message_id is the queue's safety net
- [backoff-and-jitter.md](backoff-and-jitter.md) — drain retry policy
- [dedup-on-merge.md](dedup-on-merge.md) — receiver side
- [pubnub-presence/references/dropped-connections.md](../../pubnub-presence/references/dropped-connections.md) — drain triggers
