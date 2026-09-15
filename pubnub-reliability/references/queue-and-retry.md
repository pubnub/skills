# Queue and Retry: Offline Publish Buffer

The canonical reference for buffering publishes when offline and draining them on reconnect, without losing or duplicating.

## When You Need It

- Mobile app where the user can go through a tunnel mid-action
- Field-service / IoT publisher with intermittent connectivity
- A web app where a transient network error mid-publish would otherwise drop the user's message

If publish failures are tolerable (e.g., a live ticker where stale data isn't valuable), you don't need this. For anything user-typed or critical telemetry, you do.

## Storage Choice for PubNub Offline Queue

Pick persistent storage appropriate to your platform:

| Platform | Storage |
|---|---|
| Web | IndexedDB or localStorage |
| Mobile (iOS/Android) | SQLite or platform-specific KV store |
| Never | In-memory only (lost on crash) |

The queue must survive app restart. Each queued item needs: `message_id` (for idempotency), `channel`, `message` payload, `enqueued_at`, and `attempts` counter.

## PubNub-Specific Drain Triggers

For underlying [`addListener` and `pubnub.publish`](../../pubnub-app-developer/SKILL.md) mechanics see the canonical owner.

**Critical PubNub reconnect integration:** Drain when PubNub SDK reports `PNReconnectedCategory` or `PNConnectedCategory` status. See [pubnub-presence/references/dropped-connections.md](../../pubnub-presence/references/dropped-connections.md) for PubNub's reconnect behavior.

```javascript
pubnub.addListener({
  status: (s) => {
    if (s.category === 'PNReconnectedCategory' || s.category === 'PNConnectedCategory') {
      drainOnce();  // PubNub is back online
    }
  }
});
```

Also drain on app start and on a periodic backstop interval (every 30s).

## Idempotency Is Mandatory

The queue can publish the same item twice if it crashes mid-publish (the publish succeeded, the queue write failed). The receiver must dedup by `message_id` — see [idempotent-publish.md](idempotent-publish.md). This is non-optional for PubNub offline queues.

## Observability

Log queue depth periodically. For correlation field conventions and incident triage that uses queue depth, see [pubnub-observability/references/logging-correlation.md](../../pubnub-observability/references/logging-correlation.md) and [pubnub-observability/references/incident-runbook.md](../../pubnub-observability/references/incident-runbook.md).

## Related Reading

- [idempotent-publish.md](idempotent-publish.md) — message_id is the queue's safety net
- [backoff-and-jitter.md](backoff-and-jitter.md) — drain retry policy
- [dedup-on-merge.md](dedup-on-merge.md) — receiver side
- [pubnub-presence/references/dropped-connections.md](../../pubnub-presence/references/dropped-connections.md) — drain triggers
