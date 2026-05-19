<!-- canonical-for: DROPPED_CONNECTIONS -->
<!-- used-by: pubnub-app-developer, pubnub-reliability, pubnub-history, pubnub-observability -->

# Dropped Connections and Status Categories

The canonical reference for SDK status categories that signal connection state, plus heartbeat timeout semantics and the recovery sequence.

> **Cross-references:** Built on [`addListener` and pub/sub basics](../../pubnub-app-developer/references/publish-subscribe.md). [Reconnect with backoff and jitter](../../pubnub-reliability/references/backoff-and-jitter.md) is the standard recovery pattern. [Offline catch-up via Message Persistence and fetchMessages](../../pubnub-history/references/pagination-and-ordering.md) handles missed messages on reconnect (see also [the catch-up flow](../../pubnub-history/references/offline-catch-up.md)). [Incident triage](../../pubnub-observability/references/incident-runbook.md) for "subscribe stops receiving" depends on these categories. Heartbeat affects [transaction count / billing](../../pubnub-observability/references/usage-metrics.md). New status categories may appear after [SDK upgrades](../../pubnub-app-developer/references/sdk-upgrades.md).

## The Status Listener

All connection-state events arrive on the `status` listener:

```javascript
pubnub.addListener({
  status: (s) => {
    handleStatus(s);
  }
});
```

The `s.category` field identifies what happened.

## Status Category Catalog

| Category | Meaning | Action |
|---|---|---|
| `PNConnectedCategory` | First successful connection or new channel subscription succeeded | Reset retry counters; UI: "Connected" |
| `PNReconnectedCategory` | Recovered after a disconnect | Trigger [offline catch-up](../../pubnub-history/references/offline-catch-up.md); reset retry counters |
| `PNDisconnectedCategory` | Explicit disconnect (`pubnub.disconnect()`) | Update UI to "Offline" (intentional) |
| `PNNetworkDownCategory` | Detected loss of network connectivity | Schedule reconnect with [backoff](../../pubnub-reliability/references/backoff-and-jitter.md); UI: "Reconnecting…" |
| `PNNetworkUpCategory` | Network came back; reconnect imminent | Optional: optimistic UI |
| `PNTimeoutCategory` | Subscribe call timed out (long-poll) | Normal — SDK auto-reconnects; log only if frequent |
| `PNAccessDeniedCategory` | Access Manager denied the operation | Refresh [token](../../pubnub-security/references/access-manager.md); re-subscribe |
| `PNBadRequestCategory` | Malformed request | Bug; investigate the call that triggered it |
| `PNDecryptionErrorCategory` | Could not decrypt incoming message | Verify [cipherKey / cryptoModule](../../pubnub-security/references/encryption.md) match between publisher and subscriber |
| `PNUnknownCategory` | Unrecognized status | Log and continue; check for SDK upgrade |
| `PNRequestMessageCountExceededCategory` | Subscribe got > messageCountThreshold messages in one batch | Catch-up window may be too long; reduce or paginate from history |

## Heartbeat Timeout

Presence's `timeout` action fires when a client misses heartbeats. Default heartbeat interval: **300 seconds**.

| Heartbeat config | Trade-off |
|---|---|
| 30s | Fast leave detection; high transaction count |
| 60s | Reasonable middle for chat |
| 120–180s | Lower cost; ~2–3 min stale presence |
| 300s (default) | Lowest cost; ~5 min stale presence |

A timeout means the **server** decided the client is gone. The client itself may still believe it's connected (e.g., put-to-sleep mobile app).

## The Recovery Sequence

A typical disconnect-and-recover sequence:

```
T0: Client is connected (PNConnectedCategory previously)
T1: Network drops
T2: PNNetworkDownCategory fired
       → reconnectController.scheduleReconnect() with backoff+jitter
T3: Network restored
T4: PNNetworkUpCategory fired (optional)
T5: SDK reconnects on its own schedule
T6: PNReconnectedCategory fired
       → reset retry counters
       → trigger offline catch-up via fetchMessages from last seen timetoken
       → resume normal operation
```

If the disconnect lasts longer than the [short-term cache window](../../pubnub-history/references/offline-catch-up.md), automatic catch-up will miss messages. Use Message Persistence + explicit fetch.

## Network-Down vs Timeout

| Symptom | Cause |
|---|---|
| Repeated `PNTimeoutCategory` with no network down | Long-poll timeout; usually benign — SDK reconnects |
| Single `PNNetworkDownCategory` | Real network loss; expect `PNReconnectedCategory` later |
| `PNNetworkDownCategory` repeating without `PNReconnectedCategory` | Persistent connectivity issue; show user "still trying" |
| `presence: timeout` events for many users at once | Server-side regional issue, or clients sleeping |

## App Lifecycle on Mobile

Mobile apps go through specific lifecycle states that affect presence:

| State | Behavior |
|---|---|
| Background | iOS suspends after ~10s; Android keeps connection longer but eventually backgrounds |
| Foreground | Resume connection; expect `PNReconnectedCategory` |
| Killed | Connection dies; presence will time out per heartbeat |
| Push notification wake | Brief execution window; may briefly reconnect |

For a fully reliable mobile app:

- Use a shorter heartbeat (60–120s) so timeouts happen faster on background
- On `PNReconnectedCategory`, run [offline catch-up](../../pubnub-history/references/offline-catch-up.md)
- Persist last-seen timetoken before going to background

## Browser Tab Lifecycle

Browser tabs trigger similar transitions:

| Event | Behavior |
|---|---|
| Tab backgrounded | Connection often kept alive |
| Tab discarded (memory pressure) | Connection killed |
| `beforeunload` / `pagehide` | Last chance to call `pubnub.unsubscribeAll()` for graceful leave |
| `online` / `offline` events | Cross-check against PubNub status; usually agree |

```javascript
window.addEventListener('beforeunload', () => {
  pubnub.unsubscribeAll();
  // PubNub also auto-fires unsubscribe on disconnect; this is a graceful hint
});
```

## Anti-Patterns

| Anti-pattern | Fix |
|---|---|
| Ignoring `status` events | Always handle; 80% of "messages stopped" debugging starts here |
| Treating `PNTimeoutCategory` as an error | It's normal long-poll behavior |
| Resetting retry counters on `PNNetworkDownCategory` | Reset only on `PNConnectedCategory` / `PNReconnectedCategory` |
| Not running offline catch-up after `PNReconnectedCategory` | Run it for any disconnect that lasted longer than the short-term cache window |
| Same heartbeat for mobile and web (300s) | Tune by app — shorter for mobile is usually better |
| Counting timeouts as user "leave" actions in your UI | Possibly true, but distinguish from explicit leaves |

## Related Reading

- [presence-events.md](presence-events.md) — join / leave / timeout / state-change events
- [presence-setup.md](presence-setup.md) — heartbeat configuration
- [multi-device-sync.md](multi-device-sync.md) — when the same user has multiple connections
- [pubnub-reliability/references/backoff-and-jitter.md](../../pubnub-reliability/references/backoff-and-jitter.md) — reconnect strategy
- [pubnub-history/references/offline-catch-up.md](../../pubnub-history/references/offline-catch-up.md) — catch up after disconnect
