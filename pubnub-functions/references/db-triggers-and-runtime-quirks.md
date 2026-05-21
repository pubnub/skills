<!-- canonical-for: FUNCTION_DB_TRIGGERS, FUNCTION_RUNTIME_QUIRKS -->
<!-- used-by: -->

> **Cross-references:** For pure routing to external systems prefer [Events & Actions action targets](../../pubnub-events-and-actions/references/event-types.md) (see also [the action-targets reference](../../pubnub-events-and-actions/references/action-targets.md)). For [retry/backoff and queue-and-retry](../../pubnub-reliability/references/queue-and-retry.md) outside the Function. [Vault for credentials](../../pubnub-keyset-management/references/key-rotation-and-hygiene.md). [Logging correlation fields](../../pubnub-observability/references/logging-correlation.md).

# DB Triggers and Runtime Quirks

This document covers two related topics:
1. Using a Function as a **DB trigger** to mirror PubNub messages into an external store.
2. **Runtime quirks** of the Functions environment that bite first-time users.

---

## Part 1 — DB Trigger Pattern

A common Functions use case is "every message on `audit.*` should also go into Postgres / Mongo / DynamoDB". This is a DB trigger.

### Decision: Function vs Events & Actions

| Question | Use Function | Use Events & Actions |
|---|---|---|
| Need to transform / redact before write? | Yes | No |
| Need conditional routing per message? | Yes | Yes (filters) |
| Pure write, no transform, no decision? | No | **Yes** |
| Need batching to S3 or Kinesis? | No | **Yes** |
| Need < 50ms write SLA? | Yes | No (E&A is async) |

If in doubt, prefer [Events & Actions](../../pubnub-events-and-actions/SKILL.md) — it has retries, batching, and DLQs built in. Use a Function only when you need transformation or write-on-publish coupling.

### After Publish DB Trigger

```javascript
export default async (request) => {
  const xhr = require('xhr');
  const vault = require('vault');

  try {
    const apiKey = await vault.get('db_writer_key');
    const payload = {
      channel: request.channels[0],
      message_id: request.message.id,
      user_id: request.message.user_id,
      timetoken: request.message.timetoken,
      body: request.message
    };

    await xhr.fetch('https://db-writer.example.com/insert', {
      method: 'POST',
      headers: { 'Authorization': `Bearer ${apiKey}` },
      body: JSON.stringify(payload)
    });

    return request.ok();
  } catch (err) {
    console.error('DB write failed', err);
    return request.ok();
  }
};
```

Notes:
- **Always include the four correlation fields** (channel, message id, user id, timetoken) so the row can be traced back to PubNub history.
- Return `request.ok()` even on DB failure for After Publish; the message has already been delivered to subscribers, and aborting changes nothing.
- For Before Publish DB triggers, decide whether DB failure should block the publish (`request.abort()`) or not (`request.ok()`).

### Idempotency for DB Triggers

Functions execute **at-least-once** under retry. Without idempotency, a network hiccup can double-write a row.

Two options:
1. **Use the message id as the DB primary key** (or a unique constraint). Duplicate inserts then fail-safe.
2. **Server-side dedup using KVStore** (see the [idempotent publish owner](../../pubnub-reliability/references/idempotent-publish.md)).

---

## Part 2 — Runtime Quirks

These are non-obvious behaviors of the Functions runtime. Each has bitten production users.

### Quirk 1: Cold Start

A Function that hasn't run in a while takes ~100–300ms longer on the first invocation. Don't put cold-sensitive code paths in Before Publish on a low-traffic channel.

**Mitigation:** Set up an On Interval Function that pings the cold endpoint every 5 minutes, OR move the work to After Publish (delay isn't user-visible).

### Quirk 2: 3-Call Cap on XHR + PubNub API Calls

A single Function execution is capped at **3 external calls** — counting **only `xhr.fetch(...)` and PubNub API invocations** from the `pubnub` module (`publish`, `signal`, `fire`, `grant`, etc.). The 4th call raises an `"execution calls exceeds"` error and aborts the handler.

**What counts toward the 3-call cap:**
- `xhr.fetch(...)` — any HTTP request via the `xhr` module
- `pubnub.publish(...)`, `pubnub.signal(...)`, `pubnub.fire(...)`
- Other `pubnub` module API calls (grants, presence, etc.)

**What does NOT count toward the 3-call cap:**
- `kvstore.get(...)`, `kvstore.set(...)`, `kvstore.removeItem(...)`, `kvstore.incrCounter(...)` — all KVStore ops are local to the runtime and free of this cap.
- `vault.get(...)` — vault is local; vault has its **own** 10-call-per-execution limit (see [Quirk 10](#quirk-10-vault-may-be-unavailable-or-return-not-found) and the [Vault Module section](functions-modules.md#vault-module)).
- `console.log`, `console.error`.
- Pure CPU helpers: `crypto`, `jwt`, `uuid`, `utils`, `advanced_math`, `jsonpath`, `codec/*`.

**Configurable.** The 3-call cap can be **raised by request via PubNub Support** if your application legitimately needs more external calls per invocation.

**Mitigation:**
- For HTTP fan-out, replace N fetches with one fetch to a downstream service that fans out for you (a small Lambda / Cloud Function / SQS+worker).
- For chain-level designs, prefer **3 hops × 3 calls = 9 calls** spread across the chain over jamming everything into one Function. See [Chaining vs Forking](functions-chaining.md).
- Use the `pubnub` module's `fire(...)` for analytics side-effects rather than `publish(...)` — same 1-call cost, but `fire` skips subscriber delivery if all you want is to trigger another Function.

### Quirk 3: No Top-Level `await`

Module imports must be at top level, but `await` inside the module body is not allowed. Wrap initialization in the handler.

```javascript
const db = require('kvstore');

let cachedConfig = null;

export default async (request) => {
  if (!cachedConfig) {
    cachedConfig = await db.get('config');
  }
  // ...
  return request.ok();
};
```

Note: `cachedConfig` survives across invocations on the same warm worker but is cleared on cold start. Don't rely on it for correctness, only for latency.

### Quirk 4: `console.log` Truncation

Function log lines are capped at ~1KB. Long stack traces are silently chopped. Log structured JSON of the **interesting** fields, not the whole error.

```javascript
console.log(JSON.stringify({
  channel: request.channels[0],
  message_id: request.message.id,
  err_msg: err.message,
  err_code: err.code
}));
```

See the [logging correlation owner](../../pubnub-observability/references/logging-correlation.md) for the four required fields on every log.

### Quirk 5: Publishes from Functions Bypass Before-Publish Filters

If Function A publishes to channel `X`, the Before Publish on `X` runs but **does not see the original publisher's message-meta**. Plan for this when writing security checks.

### Quirk 6: KVStore TTL is Not Real-Time

KVStore TTL is best-effort eventual. Do not use TTL for security-critical expiry (e.g., one-time-use tokens). Combine TTL with an explicit expiry check in your handler.

```javascript
const entry = await db.get(`token:${id}`);
if (!entry || entry.expires_at < Date.now()) {
  return request.abort();
}
```

### Quirk 7: On Interval Drift

On Interval Functions schedule next-run from previous-run-end, not previous-run-start. A 60-second interval Function that takes 10 seconds runs at t=0, t=70, t=140… If you need wall-clock alignment, store the last-run time in KVStore and skip if too soon.

### Quirk 8: `require` is handler-scoped

`require()` is reliably defined only **inside** the handler. `globalThis.require` may be `undefined`. If helper functions you call need `require()`, capture it inside the handler and pass it in — or set `globalThis.require = require` inside the handler before calling helpers.

```javascript
// BAD: helper assumes globalThis.require
function getDb() {
  const db = require('kvstore');   // throws in some runtimes
  return db;
}

export default async (request) => {
  // ...
};
```

```javascript
// GOOD: capture `require` inside the handler, pass into helpers
function getDb(req) {
  const db = req('kvstore');
  return db;
}

export default async (request) => {
  const db = getDb(require);
  // ...
  return request.ok();
};
```

This matters most when **bundling/transpiling** (esbuild + TS). See the [bundling and TypeScript reference](bundling-and-typescript.md) for the full set of build-system implications.

### Quirk 9: `request.path` is not always a plain string

On Request handlers should treat `request.path` defensively:

- It may not be a plain `string` (some runtimes wrap it).
- It may contain **non-printable characters** in pathological cases.
- It may or may not include a leading `/` depending on routing config (see [On Request URI Routing](functions-basics.md)).
- It may carry a query-string suffix (`?...`) you didn't expect.

Normalize before matching or parsing:

```javascript
export default async (request, response) => {
  const raw  = request.path;
  const path = String(raw || '')        // force to string
    .split('?')[0]                      // strip query
    .replace(/[\x00-\x1f]/g, '');       // drop any non-printables

  // Defensive leading-slash handling
  const segments = path.replace(/^\//, '').split('/');

  // Now segments is safe to compare with literal strings
  if (segments[0] !== 'v1') {
    return response.send({ error: 'Not Found' }, 404);
  }
  // ...
};
```

The defensive normalization is also why the Single-Router Pattern in [`functions-basics.md`](functions-basics.md) opens with `String(request.path || '').split('?')[0]`.

### Quirk 10: `vault` may be unavailable or return "Not Found"

Two distinct vault failure modes show up in production:

**A. Vault module not enabled on the key.** `require('vault')` can return `undefined`, or return an object that is missing `.get()`. Guard before use:

```javascript
const vault = require('vault');

if (!vault || typeof vault.get !== 'function') {
  // Vault not enabled on this key — fail open or abort depending on use case
  console.error('vault module unavailable');
  return request.abort();
}

const apiKey = await vault.get('API_KEY');
```

**B. "Not Found" despite the key being visible in the Portal.** If `vault.get('API_KEY')` returns `"Not Found"` (or `null`) but the key clearly exists in the Admin Portal's Secrets section, the Portal entry almost certainly contains **hidden characters** — trailing whitespace, zero-width characters, or copy-paste residue from a rich-text source. Re-create the key by typing it directly into the Portal field.

**Caching tip.** Cache `vault.get(...)` results on `globalThis` to stay under the 10-call execution limit:

```javascript
export default async (request) => {
  if (!globalThis.__creds) globalThis.__creds = {};
  if (!globalThis.__creds.API_KEY) {
    const vault = require('vault');
    globalThis.__creds.API_KEY = await vault.get('API_KEY');
  }
  const apiKey = globalThis.__creds.API_KEY;
  // ...
};
```

The cache survives across invocations on the same warm worker (see Quirk 3 for the same warm-worker behavior on KVStore reads). Combined with the per-execution 10-call ceiling, you rarely hit the limit in practice.

### Quirk 11: `sendFile()` events: `request.message` may be undefined

For file publish events triggered by the SDK's `sendFile(...)` API, `request.message` can be `undefined`. The runtime emits file metadata under separate fields (file id, file name, channel), but the user-facing message body is empty unless the caller explicitly attached one.

```javascript
export default async (request) => {
  // BAD: assumes request.message exists
  const text = request.message.text;   // TypeError when triggered by sendFile()

  // GOOD: defensive
  const maybeText = request.message?.text;
  if (maybeText === undefined) {
    // Likely a file publish — handle file-event shape here, or just continue.
    return request.ok();
  }
  // ...
};
```

If you want a non-empty `request.message` on file events, pass the second `message` argument when calling `sendFile(...)` from the SDK:

```javascript
// On the publisher side (client SDK, not inside the Function)
pubnub.sendFile({
  channel: 'uploads.user-123',
  file: fileObj,
  message: { caption: 'avatar' }
});
```

When testing Functions that listen to channels carrying both regular publishes and file publishes, always include a `sendFile`-shaped test case so you catch the undefined-message path.
