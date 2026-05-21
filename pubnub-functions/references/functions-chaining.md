<!-- canonical-for: FUNCTION_CHAINING -->
<!-- used-by: -->

> **Cross-references:** Functions chaining is the *only* PubNub mechanism for transforming a message and then routing the result. For pure routing (no transform), prefer [Events & Actions](../../pubnub-events-and-actions/SKILL.md). For pub/sub semantics see the [publish-subscribe owner](../../pubnub-app-developer/references/publish-subscribe.md).

# Function Chaining

PubNub Functions can trigger one another by republishing on a channel that another Function watches. This is **chaining**.

## The 3-Chain Rule

A single inbound publish can result in **at most 3 chained Function executions**. After that, further publishes from inside a Function are silently dropped to prevent infinite loops.

Count includes:
- The Before Publish on the source channel
- A republish that triggers another Function
- A second republish that triggers a third Function

A 4th hop is dropped.

### Related limits

- **5 consecutive Functions per sequence.** Beyond the 3-hop cap, the runtime enforces a separate cap on the total number of Functions that may execute in a single sequence. Treat **5** as the practical ceiling; design for **3 hops** and you stay safely under both limits.
- **3 external calls per Function** (XHR + PubNub API invocations). Each hop in a chain still has its own 3-call cap, so a 3-hop chain can do up to 9 external calls across the chain — but each individual hop is still capped at 3. See [`functions-basics.md` "Execution Limits"](functions-basics.md) and the [3-Operation Cap quirk](db-triggers-and-runtime-quirks.md).
- **Contact PubNub Support** if your application requires more than 3 hops or more than 3 external calls per hop. Both limits are configurable on request.

## Chaining vs Forking

Both patterns republish from inside a Function. The difference is **whether you `await` the publish** before returning the completion call.

### Chaining (fire-and-forget)

The republish is fired but not awaited. The Function completes immediately, the original message continues through its pipeline, and the downstream Function eventually runs whenever the platform schedules it. Use this when:

- You don't need a result back from the downstream Function.
- You want minimal added latency on the source publish.
- You're OK with downstream-Function failure being detached from the source publish's outcome.

```javascript
export default async (request) => {
  const pubnub = require('pubnub');

  // Fire-and-forget: don't await
  pubnub.publish({ channel: 'next-step', message: request.message });

  return request.ok();   // returns immediately; the downstream Function runs async
};
```

### Forking (wait for completion)

The republish is awaited before completion. The source publish is held until the publish acknowledgment returns. Use this when:

- You need to confirm the next stage was at least accepted before returning.
- You need ordering guarantees within the source publish's response time.
- You're paying the latency cost knowingly.

```javascript
export default async (request) => {
  const pubnub = require('pubnub');

  // Wait for publish to complete
  await pubnub.publish({ channel: 'next-step', message: request.message });

  return request.ok();
};
```

**Default:** prefer **chaining (fire-and-forget)** unless you have a concrete reason to await. Forking inflates latency on the source publish for every hop and is a common cause of timeout-induced aborts.

> The `await` only confirms the downstream publish was accepted by PubNub — it does **not** wait for the downstream Function to finish. There is no "wait for the next Function to return" primitive.

## When to Chain

| Use case | Pattern |
|---|---|
| Enrich → notify | Function A enriches the message, publishes to `notify.*`; Function B sends a webhook |
| Sanitize → store | Function A redacts PII, publishes to `safe.*`; Function B writes to history |
| Fan-out by tenant | Function A reads tenant ID, republishes to `tenant.{id}.*` |

## When NOT to Chain

- **Sequential transforms on the same channel** — combine into one Function. Chaining adds latency and uses your chain budget.
- **External delivery only** — use [Events & Actions](../../pubnub-events-and-actions/SKILL.md) instead of `xhr` from a Function. E&A has retries and batching built in.
- **Threshold-triggered work** — use [Illuminate Decisions](../../pubnub-illuminate/references/decisions-4-step-workflow.md) instead of polling KVStore in an On Interval.

## Sharing State Between Chained Functions

Chained Functions are independent invocations. **`request.message` is the only payload guaranteed to flow forward** — variables, helper objects, and closure state are not preserved between hops.

For state that doesn't fit cleanly in the message envelope (or that you want kept out of the on-the-wire payload), use **`kvstore`** as the inter-Function carrier:

```javascript
// Function A (Before Publish) — compute and stash
export default async (request) => {
  const db     = require('kvstore');
  const pubnub = require('pubnub');

  // Compute something derived from this message
  const computed = {
    rate:      0.5,
    user_id:   request.message.user_id,
    updatedAt: Date.now()
  };

  // Stash with a per-message key so Function B can find it
  const stashKey = `chain:${request.message.id}`;
  await db.set(stashKey, computed, 5);   // 5-minute TTL is plenty for a chain hop

  // Continue to Function B
  pubnub.publish({ channel: 'next-step', message: { id: request.message.id } });

  return request.ok();
};

// Function B (Before Publish on `next-step`) — read
export default async (request) => {
  const db = require('kvstore');

  const stashKey = `chain:${request.message.id}`;
  const computed = await db.get(stashKey);

  if (!computed) {
    console.error('Missing chain state for', request.message.id);
    return request.abort();
  }

  // Use computed.rate, computed.user_id, etc.
  request.message.rate = computed.rate;

  return request.ok();
};
```

Design notes:

- **Use a per-message key.** `chain:${message_id}` (or `chain:${user_id}:${timetoken}`) avoids collisions across concurrent chains.
- **Use a short TTL.** Chain hops complete within seconds; a 1–5 minute TTL keeps KVStore tidy.
- **Don't store secrets** in the chain stash — they should come from `vault` in each Function independently.
- **Don't depend on TTL** for cleanup of large stashes — see [Quirk 6: KVStore TTL is Not Real-Time](db-triggers-and-runtime-quirks.md).
- KVStore reads and writes both count toward each Function's 3-call cap if you treat KVStore as a counted op (see Quirk 2 for the canonical accounting).

## Channel Namespace Hygiene

The runtime detects infinite loops and silently caps the chain at 3 hops, but **don't rely on that detection**. Avoid topology where a Function's output channel could re-trigger the same Function (directly or transitively).

### Pitfall 1: Republishing to a channel your Function listens on

```javascript
// BAD: Function listens on `events.*` AND publishes back to `events.processed`
// — which matches its own pattern.
export default async (request) => {
  const pubnub = require('pubnub');
  pubnub.publish({ channel: 'events.processed', message: { ...request.message, ok: true } });
  return request.ok();
};
```

`events.*` matches `events.processed`, so the Function's own output re-triggers it. The 3-hop cap kicks in eventually, but each chain still burns its budget.

### Pitfall 2: Wildcard subscribers downstream that point back

A second Function listening on `output.*` that publishes to `events.processed` (above) creates a transitive loop. This is harder to spot because the loop isn't visible in any single Function's code.

### Recommended pattern: distinct namespaces per stage

```javascript
// GOOD: Function A on `input.*`  publishes only to `stage1.*`
// GOOD: Function B on `stage1.*` publishes only to `stage2.*`
// GOOD: Function C on `stage2.*` publishes only to `output.*`
// — no Function ever publishes to a channel matching its own pattern.
export default async (request) => {
  const pubnub = require('pubnub');
  pubnub.publish({ channel: 'stage1.processed', message: request.message });
  return request.ok();
};
```

When in doubt, **prefix every stage's output channel with a name distinct from any active Function's pattern**. The 3-hop cap is a safety net, not a design tool.

## Ordering

Chained executions are **best-effort ordered** but not guaranteed. If ordering matters across hops, include a sequence number in the message envelope (see [schema versioning](../../pubnub-reliability/references/schema-versioning.md)).

## Debugging Chain Loops

1. In Admin Portal → Functions → My Function, watch the execution count.
2. If you see a flatline at exactly 3 executions per inbound publish, you have hit the chain cap.
3. Trace by tagging the message with a `chain_hop` counter and logging it on every Function.

## Example: 2-Hop Chain

**Function A — Before Publish on `chat.*`:**

```javascript
export default async (request) => {
  const pubnub = require('pubnub');
  request.message.enriched_at = Date.now();
  await pubnub.publish({
    channel: `notify.${request.message.user_id}`,
    message: { type: 'chat', body: request.message }
  });
  return request.ok();
};
```

**Function B — After Publish on `notify.*`:**

```javascript
export default async (request) => {
  const xhr = require('xhr');
  await xhr.fetch('https://example.com/notify', {
    method: 'POST',
    body: JSON.stringify(request.message)
  });
  return request.ok();
};
```

This counts as 2 executions out of the 3-chain budget.
