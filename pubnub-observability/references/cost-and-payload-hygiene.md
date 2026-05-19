<!-- canonical-for: PAYLOAD_HYGIENE -->
<!-- used-by: pubnub-history, pubnub-scale -->

# Cost and Payload Hygiene

The canonical reference for keeping PubNub costs predictable: payload sizing, coalescing updates, fan-out discipline, and the publish vs signal vs fire decision.

## How PubNub Pricing Works (At a High Level)

PubNub bills by **transactions**, not bytes. The dominant cost drivers in most apps:

| Cost driver | Drives this transaction count |
|---|---|
| Publishes per second | 1 per `publish()` call |
| Fan-out (N subscribers per channel) | Multiplies receive transactions |
| [Presence event](../../pubnub-presence/references/presence-events.md) volume | Joins/leaves and timers |
| History reads | 1 per `fetchMessages` request |
| Function executions | 1 per Function invocation |
| [App Context](../../pubnub-app-context/references/users.md) operations | 1 per set/get |

The single largest factor in most apps is **fan-out × publish rate**. Every other knob is downstream of those.

## Payload Sizing

### What Matters

PubNub message size affects:

- **Network bandwidth** for every subscriber (× fan-out)
- **History storage cost** if [Message Persistence](../../pubnub-history/references/pagination-and-ordering.md) is enabled
- **Mobile data plan** for client devices
- **Latency**: large messages serialize and route slower

### Targets

| Use case | Target payload |
|---|---|
| Chat message | < 1 KB |
| Game state delta | < 256 B |
| Telemetry tick | < 256 B |
| Notification | < 512 B |

PubNub's hard limit is 32 KB per message, but most apps should be far under that. Anything > 4 KB is a flag for review.

### Trim Aggressively

```javascript
// Bad — sends server-side fields the client doesn't need
const msg = {
  ...userRecord,
  ...channelRecord,
  ...auditFields,
  payload: 'hi'
};

// Good — only what the receiver needs
const msg = {
  schema_version: 1,
  message_id: crypto.randomUUID(),   // not the same as the PubNub [userId/UUID](../../pubnub-app-developer/references/sdk-patterns.md)
  sender_id: userRecord.id,
  text: 'hi'
};
```

### Files Don't Belong in `publish()`

For files (images, PDFs, audio), use the [File Sharing API](../../pubnub-chat/references/file-sharing.md), which uploads to PubNub-managed storage and delivers a reference. Stuffing base64 in `publish()` payloads is the most common cost mistake.

## Coalescing Updates

When state changes faster than a UI can render, **coalesce on the producer side**:

```javascript
class CoalescedPublisher {
  constructor(channel, intervalMs = 100) {
    this.channel = channel;
    this.intervalMs = intervalMs;
    this.pending = null;
    this.timer = null;
  }

  send(state) {
    this.pending = state;     // overwrite — latest wins
    if (!this.timer) {
      this.timer = setTimeout(() => this.flush(), this.intervalMs);
    }
  }

  async flush() {
    const s = this.pending;
    this.pending = null;
    this.timer = null;
    if (s !== null) {
      await pubnub.publish({ channel: this.channel, message: s });
    }
  }
}
```

For a cursor that moves 60 times per second, this drops publish rate to 10/s with no visible UX change.

### When to Coalesce vs Stream

| Update type | Coalesce? |
|---|---|
| Cursor / pointer position | Yes (10–30 Hz max) |
| Scroll position | Yes |
| Game tick (deltas) | Yes (30 Hz typical) |
| Chat message | No — every message matters |
| Order placement | No — every event is discrete |
| Sensor reading where only latest matters | Yes |

## Fan-Out Discipline

Fan-out is the dominant cost driver. Every `publish()` to a channel with N subscribers costs roughly N receive transactions plus 1 publish.

### Common Fan-Out Mistakes

| Mistake | Cost impact | Fix |
|---|---|---|
| Broadcasting per-user state to a room channel | Multiplies by room size unnecessarily | Use a per-user channel |
| Subscribing all clients to "all-events" channel | Every event hits every client | Use [channel groups](../../pubnub-scale/references/scaling-patterns.md) or per-topic channels |
| Replaying all history on every page load | Multiplies history reads | Cache locally, paginate |
| Echoing publishes back to the publisher | Doubles fan-out | Filter on receive (publisher_id !== self) |

### Channel-Level Architecture

Decide channel granularity by who needs to receive what:

| Granularity | Use when |
|---|---|
| Per-user (`user.123`) | Direct messages, notifications, per-user state |
| Per-room (`room.42`) | Chat rooms, multiplayer games, live events |
| Global (`broadcast`) | Announcements (be very cautious) |

## Publish vs Signal vs Fire

Three publish-like operations with different semantics and pricing tiers:

| Op | Persisted? | Triggers Functions? | Use for |
|---|---|---|---|
| `publish()` | Yes (if Message Persistence on) | Yes | Standard messages |
| `signal()` | No | Yes | Ephemeral signals (typing indicators, presence-like updates); cheaper than [`pubnub.publish()`](../../pubnub-app-developer/references/publish-subscribe.md), smaller max payload (~64 bytes) |
| `fire()` | No | Yes | Triggering server-side Functions only; not delivered to subscribers |

When the receiver doesn't need to read the data later, `signal()` is cheaper than `publish()`. For [Functions](../../pubnub-functions/references/functions-basics.md) triggers without delivery, `fire()`.

## Auditing Cost Before Shipping

Before any new feature ships, walk through:

1. **Publish rate**: how many publishes per second per active user?
2. **Fan-out**: per channel, how many subscribers?
3. **History rate**: how often are history reads triggered?
4. **Function invocations**: how many per publish?
5. **App Context operations**: how often per session?

Multiply by your projected DAU. If the projected transaction count exceeds your plan, reduce coalescing window, narrow channels, or move some work to the client.

Use [`get_pubnub_usage_metrics`](usage-metrics.md) to verify projections after launch.

## Anti-Patterns

| Anti-pattern | Fix |
|---|---|
| Files in `publish()` payloads | [File Sharing API](../../pubnub-chat/references/file-sharing.md) |
| Per-keystroke publish for typing indicators | Coalesce; or use `signal()` |
| Echoing publishes back to the publisher | Filter on receive (publisher_id !== self) |
| Subscribe-to-everything pattern | Per-topic channels + [channel groups](../../pubnub-scale/references/scaling-patterns.md) |
| `publish()` for fire-only Function triggers | `fire()` |
| `publish()` for ephemeral signals | `signal()` (also smaller payload limit) |
| 60 Hz state stream when UI renders at 10 Hz | Coalesce |
| No cost projection before launch | Walk through the 5-step audit above |

## Related Reading

- [usage-metrics.md](usage-metrics.md) — verify cost projections in production
- [test-pyramid.md](test-pyramid.md) — load tests pre-validate cost
- [pubnub-scale/references/scaling-patterns.md](../../pubnub-scale/references/scaling-patterns.md) — channel groups and wildcard subscribe for narrow fan-out
- [pubnub-history/references/retention-and-storage.md](../../pubnub-history/references/retention-and-storage.md) — selective storage, retention impact
