<!-- canonical-for: IDEMPOTENT_PUBLISH -->
<!-- used-by: pubnub-app-developer, pubnub-history, pubnub-functions, pubnub-observability -->

# Idempotent Publish

The canonical reference for client-generated message IDs that make `publish()` retries safe.

## Why Idempotent Publish

A `publish()` call can fail in three ways:

1. **Network error before the message reaches PubNub** — safe to retry.
2. **PubNub received but did not ack** — retry creates a duplicate.
3. **Ack was sent but lost** — retry creates a duplicate.

Without idempotency, your retry strategy duplicates messages whenever PubNub successfully received but the ack was lost. That's not a hypothetical — it's the standard distributed-systems failure mode.

The fix: every publish carries a **client-generated message ID**. Servers (and other clients) can detect and drop the duplicate. The producer can retry freely.

## The Pattern

### 1. Generate a Unique ID Per Logical Message

Generate the ID **once** at the moment of message creation (not at the moment of send). The same ID flows through every retry attempt.

```javascript
function createMessage(payload) {
  return {
    schema_version: 1,
    message_id: crypto.randomUUID(),    // RFC 4122 UUID v4
    sent_at: Date.now(),
    payload
  };
}

const msg = createMessage({ text: 'Hello!' });
// msg.message_id is now stable across retries
```

For schema_version semantics see [schema-versioning.md](schema-versioning.md).

### 2. Retry Carries the Same ID

```javascript
async function publishWithRetry(channel, msg, maxAttempts = 5) {
  for (let attempt = 0; attempt < maxAttempts; attempt++) {
    try {
      return await pubnub.publish({ channel, message: msg });
    } catch (err) {
      if (attempt === maxAttempts - 1) throw err;
      await sleepWithJitter(attempt);   // see backoff-and-jitter.md
    }
  }
}
```

For [backoff-and-jitter.md](backoff-and-jitter.md) details see the canonical owner.

### 3. Subscribers Dedup by `message_id`

For [`addListener` and `pubnub.subscribe`/`pubnub.publish` basics](../../pubnub-app-developer/references/publish-subscribe.md) see the canonical owner.

```javascript
const seenIds = new LRU({ max: 10_000 });

pubnub.addListener({
  message: (e) => {
    const id = e.message?.message_id;
    if (id && seenIds.has(id)) {
      return;   // duplicate; drop
    }
    if (id) seenIds.set(id, true);
    process(e.message);
  }
});
```

For dedup mechanics in detail see [dedup-on-merge.md](dedup-on-merge.md).

## ID Choice

| ID source | Pros | Cons |
|---|---|---|
| `crypto.randomUUID()` v4 | Standard, collision-free, 16 bytes encoded as 36 chars (note: distinct from PubNub's [userId/UUID](../../pubnub-app-developer/references/sdk-patterns.md)) | A bit long |
| Snowflake-style (timestamp + node + counter) | Sortable, more compact | Needs node ID assignment |
| Hash of (userId + timestamp + nonce) | Deterministic if you need the same id from another writer | Requires shared logic |
| Sequential counter | Sortable, compact | Single-writer only; resets are dangerous |

For most use cases: **UUID v4 via `crypto.randomUUID()`**.

## Server-Side Idempotency Check (PubNub Function)

If subscribers are untrusted, you can also dedup at the edge with a Function that consults a KV store. For [Function basics](../../pubnub-functions/references/functions-basics.md) and [KV Store / `require('kvstore')`](../../pubnub-functions/references/functions-modules.md) see the canonical owners.

```javascript
// before publish handler
const kvstore = require('kvstore');

export default (request) => {
  const id = request.message?.message_id;
  if (!id) return request.ok();

  const key = `dedup:${id}`;
  return kvstore.get(key).then(seen => {
    if (seen) return request.abort();           // duplicate; drop
    return kvstore.set(key, '1', 60 * 24)      // remember 1 day
      .then(() => request.ok());
  });
};
```

This adds a per-publish KV round-trip. Use only when client-side dedup is insufficient (untrusted publishers, server-side audit requirements, third-party integrations).

## Idempotency Window

`message_id` dedup is most useful within a bounded time window:

- Live subscribers: window = the `seenIds` cache TTL (typically the active session)
- History fetch + live merge: window = the bounded fetch + live overlap (minutes to an hour)
- Server-side dedup: explicit TTL on the KV entry (above example uses 24 hours)

A 24-hour idempotency window covers virtually all retry scenarios. Going longer wastes storage; going shorter risks missing slow retries from a mobile app that woke up later.

## Anti-Patterns

| Anti-pattern | Fix |
|---|---|
| Generating a new UUID inside `publishWithRetry` for each attempt | Generate once outside the retry loop |
| Using a timestamp alone as the ID | Two messages in the same ms collide; use UUID + timestamp |
| Server-only dedup, no client ID | Server can't know which incoming message is a retry without an ID |
| Putting the ID outside the message envelope (e.g., as a custom header) | History fetch returns just the message body — the ID must travel with it |
| Forgetting `seenIds` on the receiver and relying entirely on PubNub | PubNub does not dedup; that's the producer's job |

## Verification

Test idempotency with a controlled retry:

```javascript
const msg = createMessage({ text: 'Test' });

// Publish 3 times with the SAME message_id
await pubnub.publish({ channel: 'test', message: msg });
await pubnub.publish({ channel: 'test', message: msg });
await pubnub.publish({ channel: 'test', message: msg });

// Subscribe and confirm only one is processed
let processedCount = 0;
const seenIds = new Set();
pubnub.addListener({
  message: (e) => {
    if (seenIds.has(e.message.message_id)) return;
    seenIds.add(e.message.message_id);
    processedCount++;
  }
});
pubnub.subscribe({ channels: ['test'] });

setTimeout(() => console.assert(processedCount === 1), 5000);
```

## Related Reading

- [backoff-and-jitter.md](backoff-and-jitter.md) — the retry that idempotency makes safe
- [dedup-on-merge.md](dedup-on-merge.md) — receiver-side dedup logic
- [queue-and-retry.md](queue-and-retry.md) — offline queue uses message_id as key
- [schema-versioning.md](schema-versioning.md) — envelope contains both `schema_version` and `message_id`
