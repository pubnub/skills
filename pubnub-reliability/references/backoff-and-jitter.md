<!-- canonical-for: BACKOFF_AND_JITTER -->
<!-- used-by: pubnub-app-developer, pubnub-presence, pubnub-history, pubnub-observability -->

# Reconnect: Backoff and Jitter

The canonical reference for retry-with-backoff in PubNub clients and any wrappers around them.

## Why Backoff + Jitter

When a regional outage ends and thousands of clients reconnect simultaneously, a pure "retry every N seconds" strategy creates a thundering herd that immediately overloads the recovering infrastructure.

Two compounding mitigations:

- **Exponential backoff**: each retry waits longer than the previous one
- **Jitter**: each retry adds randomness, breaking the synchronization across clients

## The Algorithm: Exponential Backoff with Full Jitter

```javascript
function nextRetryDelay(attemptNumber, opts = {}) {
  const initialMs = opts.initialMs ?? 200;
  const maxMs     = opts.maxMs ?? 30_000;
  const multiplier = opts.multiplier ?? 2;

  const expDelay = Math.min(maxMs, initialMs * Math.pow(multiplier, attemptNumber));
  const jittered = Math.random() * expDelay;   // full jitter: [0, expDelay)
  return jittered;
}
```

Recommended defaults:

| Param | Value | Rationale |
|---|---|---|
| `initialMs` | 200 | Fast recovery from a true blip |
| `maxMs` | 30000 | Cap so we don't wait forever |
| `multiplier` | 2 | Standard exponential growth |
| jitter | full | Better than equal jitter for synchronization-breaking |

### Attempt Number Sequence

| Attempt | Without jitter | With full jitter (range) |
|---|---|---|
| 1 | 200ms | 0–200ms |
| 2 | 400ms | 0–400ms |
| 3 | 800ms | 0–800ms |
| 4 | 1.6s | 0–1.6s |
| 5 | 3.2s | 0–3.2s |
| 6 | 6.4s | 0–6.4s |
| 7 | 12.8s | 0–12.8s |
| 8 | 25.6s | 0–25.6s |
| 9+ | capped at 30s | 0–30s |

## Bounded Retries

Don't retry forever. After a sensible cap, escalate to the user (show a "still trying to reconnect" UI) or surface to monitoring.

```javascript
const MAX_ATTEMPTS = 12;     // ~5 min cumulative with capped backoff

class ReconnectController {
  constructor(opts = {}) {
    this.opts = opts;
    this.attempts = 0;
    this.timer = null;
  }

  scheduleReconnect(reconnectFn, onGiveUp) {
    if (this.attempts >= MAX_ATTEMPTS) {
      onGiveUp();
      return;
    }
    const delay = nextRetryDelay(this.attempts, this.opts);
    this.attempts++;
    this.timer = setTimeout(reconnectFn, delay);
  }

  reset() {
    this.attempts = 0;
    if (this.timer) clearTimeout(this.timer);
  }
}
```

## Listening for PubNub Connection State

Drive the controller from the SDK's connection-state events. For [`PNNetworkDownCategory` / `PNNetworkUpCategory` / `PNReconnectedCategory` semantics](../../pubnub-presence/references/dropped-connections.md) see the canonical owner. The [`addListener` mechanics](../../pubnub-app-developer/references/publish-subscribe.md) are documented separately.

```javascript
const reconnectController = new ReconnectController();

pubnub.addListener({
  status: (s) => {
    switch (s.category) {
      case 'PNConnectedCategory':
      case 'PNReconnectedCategory':
        reconnectController.reset();
        onConnected();
        break;

      case 'PNNetworkDownCategory':
      case 'PNNetworkIssuesCategory':
      case 'PNTimeoutCategory':
        reconnectController.scheduleReconnect(
          () => pubnub.reconnect(),
          () => showUserGiveUpUI()
        );
        break;
    }
  }
});
```

## SDK-Level Retry Knobs

Some PubNub SDKs expose their own retry configuration. When present, use them rather than wrapping. Examples vary by SDK; check your SDK docs for:

- `retryConfiguration` (JavaScript)
- `requestRetryConfig` (Swift)
- Reconnection policy (Java, Kotlin)

The defaults are usually reasonable for chat / consumer apps. For high-frequency IoT or finance, tune narrower.

## Per-Operation vs Per-Connection Backoff

| Failure type | Use this backoff |
|---|---|
| Subscribe / connection drop | Connection-level backoff (this document) |
| Single `publish` failure | Per-publish retry with idempotent message ID — see [idempotent-publish.md](idempotent-publish.md) |
| Single [`fetchMessages`](../../pubnub-history/references/pagination-and-ordering.md) failure | Wrap the call in retry with backoff; cap attempts at 3 |

Don't conflate them. A connection-level backoff that pauses publishes too will starve the queue.

## Backoff for Other Calls

Apply the same algorithm to any retried call. A simple wrapper:

```javascript
async function withBackoff(fn, opts = {}) {
  const maxAttempts = opts.maxAttempts ?? 5;
  for (let attempt = 0; attempt < maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (err) {
      if (attempt === maxAttempts - 1) throw err;
      const delay = nextRetryDelay(attempt, opts);
      await new Promise(r => setTimeout(r, delay));
    }
  }
}

// Usage
const result = await withBackoff(() => pubnub.fetchMessages({ channels: ['x'], count: 10 }));
```

## What Not to Do

| Anti-pattern | Why it fails |
|---|---|
| Fixed-interval retry (every 5s) | Thundering herd after outage |
| Exponential backoff without jitter | Still synchronized — every client's 4th attempt fires at the same moment |
| Unbounded retries | Burns battery, fills logs, never tells the user |
| Per-call retry inside a higher-level retry | Multiplies total delay; surface failures up |
| Resetting `attempts = 0` on a network blip | Loses backoff state; resets the curve too eagerly. Reset only on `PNConnectedCategory` / `PNReconnectedCategory`. |
| Sharing one `attempts` counter across multiple PubNub instances | Per-instance state |

## Related Reading

- [idempotent-publish.md](idempotent-publish.md) — make per-publish retries safe
- [queue-and-retry.md](queue-and-retry.md) — pair with offline buffering
- [pubnub-presence/references/dropped-connections.md](../../pubnub-presence/references/dropped-connections.md) — network-down event semantics
- [pubnub-observability/references/incident-runbook.md](../../pubnub-observability/references/incident-runbook.md) — incident response uses backoff observation
