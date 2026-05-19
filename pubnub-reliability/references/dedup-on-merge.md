<!-- canonical-for: DEDUP_ON_MERGE -->
<!-- used-by: pubnub-history, pubnub-illuminate, pubnub-observability -->

# Dedup on Merge

The canonical reference for deduplicating messages when combining live subscription with historical fetch, or when the same message could arrive twice via the network.

## When You Need It

| Scenario | Dedup needed? |
|---|---|
| Live subscription only, no retries | No — PubNub guarantees at-most-once per subscriber for a single connection |
| Live subscription with idempotent retry on the producer | Yes — same message_id can arrive twice if a producer retries successfully |
| Live + history merge (offline catch-up) | Yes — overlap window almost guarantees duplicates |
| Multiple PubNub instances on the same [userId](../../pubnub-app-developer/references/sdk-patterns.md) ([multi-device](../../pubnub-presence/references/multi-device-sync.md)) | No — each device is a distinct subscription stream; the user-level dedup is a UI concern |
| [Function chaining](../../pubnub-functions/references/functions-chaining.md) where a Function publishes downstream | Yes if the Function might retry |

## Dedup Key Choice

| Key | When to use | Notes |
|---|---|---|
| `message_id` (client-generated) | Producer is idempotent | Best — survives retries, history fetches, anything |
| `timetoken` | Producer is not idempotent but you only care about this connection | Unique per publish, but a retry creates a new timetoken |
| Hash of payload | No `message_id` and no other identifier | Fragile; identical-payload messages dedup as one |

**Always prefer `message_id`** when the producer is under your control. See [idempotent-publish.md](idempotent-publish.md) for how to generate it.

## Set vs LRU vs TTL

### Small Window: `Set`

For short-lived dedup (a single session, a single catch-up pass):

```javascript
const seen = new Set();

function process(msg) {
  const key = msg.message_id ?? msg.timetoken;
  if (seen.has(key)) return;
  seen.add(key);
  render(msg);
}
```

Simple but unbounded — fine for a session of minutes.

### Bounded: LRU

For longer sessions, cap memory with an LRU:

```javascript
class LRU {
  constructor(max = 10_000) {
    this.max = max;
    this.map = new Map();
  }
  has(key) {
    if (!this.map.has(key)) return false;
    const v = this.map.get(key);
    this.map.delete(key);
    this.map.set(key, v);   // refresh recency
    return true;
  }
  add(key) {
    if (this.map.has(key)) this.map.delete(key);
    this.map.set(key, true);
    if (this.map.size > this.max) {
      this.map.delete(this.map.keys().next().value);
    }
  }
}

const seen = new LRU(10_000);
function process(msg) {
  const key = msg.message_id ?? msg.timetoken;
  if (seen.has(key)) return;
  seen.add(key);
  render(msg);
}
```

10,000 entries covers most chat / consumer apps. Adjust based on traffic.

### Time-Bounded: TTL Map

For long-running services where memory pressure dominates:

```javascript
class TTLMap {
  constructor(ttlMs = 24 * 60 * 60 * 1000) {
    this.ttlMs = ttlMs;
    this.map = new Map();
  }
  has(key) {
    const exp = this.map.get(key);
    if (!exp) return false;
    if (exp < Date.now()) {
      this.map.delete(key);
      return false;
    }
    return true;
  }
  add(key) { this.map.set(key, Date.now() + this.ttlMs); }
}
```

Pair with periodic GC if `add` rate is high.

## Live + History Merge

The single most common dedup scenario. See [pubnub-history/references/offline-catch-up.md](../../pubnub-history/references/offline-catch-up.md) for the full flow. The critical merge pattern:

```javascript
const seen = new LRU(10_000);

function process(msg) {
  const key = msg.message_id ?? msg.timetoken;
  if (seen.has(key)) return;
  seen.add(key);
  render(msg);
}

// 1. Catch up first
const caughtUp = await fetchAllMissed(channel, lastTT);
for (const msg of caughtUp) process(msg);

// 2. Subscribe live (see [addListener / pubnub.subscribe basics](../../pubnub-app-developer/references/publish-subscribe.md))
pubnub.addListener({ message: e => process(e.message ? e : { ...e, message: e.message }) });
pubnub.subscribe({ channels: [channel] });
```

Order matters: process caught-up messages first (they're older), then start consuming live. The dedup `Set` already contains everything from the catch-up, so any live message that overlaps is dropped.

## Cross-Channel vs Per-Channel Dedup

Choose based on whether duplicates can cross channels:

| Choice | When |
|---|---|
| One global dedup set | Producer publishes the same `message_id` to multiple channels (rare; usually a smell) |
| Per-channel dedup set | Standard case |
| Per-(channel, user) dedup set | Multi-tenant where channels can be hit by different `message_id` namespaces |

Default to per-channel.

## Persistent Dedup (Across App Restarts)

For mobile apps that should not re-process messages already shown after a cold start:

```javascript
async function process(msg) {
  const key = msg.message_id ?? msg.timetoken;
  if (await persistentSeen.has(channel, key)) return;
  await persistentSeen.add(channel, key);
  render(msg);
}
```

Back the persistent set with localStorage, IndexedDB, SQLite, etc. **Bound it** — purge entries older than your offline window (1 day, 1 week).

## Anti-Patterns

| Anti-pattern | Fix |
|---|---|
| Unbounded in-memory `Set` for a long-running process | Use LRU or TTL |
| Dedup by `JSON.stringify(payload)` | Slow; brittle; identical-payload messages collapse |
| Live-only dedup that doesn't outlive the connection | Persist to local storage if app restarts must not reprocess |
| Dedup the message before the catch-up pass populates the set | Process catch-up first, live second |
| Treating PubNub as if it dedupes | PubNub does not dedup; it's the consumer's job |
| One global set across all channels and users | Per-channel (or per-(channel, user)) instead |

## Related Reading

- [idempotent-publish.md](idempotent-publish.md) — the producer side
- [queue-and-retry.md](queue-and-retry.md) — offline queue keyed by message_id
- [pubnub-history/references/offline-catch-up.md](../../pubnub-history/references/offline-catch-up.md) — the merge flow that needs dedup
