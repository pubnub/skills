<!-- canonical-for: FUNCTION_BASICS -->
<!-- used-by: -->

> **Cross-references:** Built on [pub/sub semantics](../../pubnub-app-developer/references/publish-subscribe.md) and [SDK initialization including userId/UUID](../../pubnub-app-developer/references/sdk-patterns.md). For [chained Functions](functions-chaining.md) and [DB-trigger patterns + runtime quirks](db-triggers-and-runtime-quirks.md). For non-transform routing prefer [Events & Actions](../../pubnub-events-and-actions/SKILL.md).

# PubNub Functions 2.0 Basics

## Overview

PubNub Functions 2.0 execute serverless JavaScript code at the network edge in response to events. Key features:

- **Event-driven**: Trigger on messages, HTTP requests, or schedules
- **Serverless**: No infrastructure to manage
- **Edge execution**: Low latency, close to users
- **Built-in modules**: KVStore, XHR, Vault, Crypto, and more

## Function Types

### 1. Before/After Publish (Message Events)

Trigger when messages are published to matching channels.

```javascript
// Before Publish - Can modify or block messages
export default async (request) => {
  try {
    console.log('Channel:', request.channels[0]);
    console.log('Message:', request.message);

    // Modify message
    request.message.processedAt = new Date().toISOString();

    // Allow message to proceed (modified)
    return request.ok();

    // Or block the message
    // return request.abort();
  } catch (error) {
    console.error('Error:', error);
    return request.abort();
  }
};
```

**Key Methods:**
- `request.ok()` - Allow message to proceed
- `request.abort()` - Block message from delivery
- `request.message` - Access/modify message payload
- `request.channels` - Array of channel names

### 2. On Request (HTTP Endpoint)

Trigger when HTTP requests hit the function's URL.

```javascript
export default async (request, response) => {
  try {
    // Access request data
    const method = request.method;      // GET, POST, PUT, DELETE
    const params = request.params;      // URL path parameters
    const query = request.query;        // Query string parameters
    const body = await request.json();  // Parse JSON body

    // Set CORS headers
    response.headers['Access-Control-Allow-Origin'] = '*';
    response.headers['Access-Control-Allow-Methods'] = 'GET, POST';

    // Handle OPTIONS for CORS preflight
    if (method === 'OPTIONS') {
      return response.send('', 200);
    }

    // Process and respond
    return response.send({ status: 'success', data: body }, 200);
  } catch (error) {
    console.error('Error:', error);
    return response.send({ error: 'Internal server error' }, 500);
  }
};
```

**Key Methods:**
- `response.send(body, statusCode)` - Send HTTP response
- `request.json()` - Parse request body as JSON
- `request.method` - HTTP method
- `request.params` - URL path parameters
- `request.query` - Query string parameters

### 3. On Interval (Scheduled)

Trigger on a schedule (e.g., every 5 minutes).

```javascript
export default async (event) => {
  const db = require('kvstore');
  const pubnub = require('pubnub');

  try {
    console.log('Scheduled execution at:', new Date().toISOString());

    // Perform periodic tasks
    const stats = await db.get('daily_stats');

    // Publish aggregated data
    await pubnub.publish({
      channel: 'analytics',
      message: { type: 'daily_summary', data: stats }
    });

    return event.ok();
  } catch (error) {
    console.error('Scheduled task error:', error);
    return event.abort();
  }
};
```

**Key Methods:**
- `event.ok()` - Signal successful execution
- `event.abort()` - Signal execution failure

## Channel Patterns (Wildcards)

Configure which channels trigger the function.

### Rules

- Wildcard (`*`) must be at the **end**
- Maximum **two literal segments** before wildcard
- Period (`.`) is the delimiter

### Valid Patterns

```
alerts.*           → matches alerts.critical, alerts.info
user.notifications.* → matches user.notifications.email
chat.*             → matches chat.room1, chat.general
```

### Invalid Patterns

```
alerts.*.critical  → wildcard not at end
*.notifications    → wildcard at start
a.b.c.d.*          → too many segments
```

## On Request URI Routing

Channel patterns (above) gate which messages trigger Before/After Publish Functions. **On Request Functions use a separate routing mechanism: URI path matching against the configured pattern.** The rules are different from channel-pattern wildcards.

### Rules

- Routes are matched by a **character-by-character comparison** of the incoming URI path against the configured pattern.
- `*` matches **0..N** characters. **Multiple `*`** are allowed within a single pattern.
- Slashes are not special. **Trailing slashes must match exactly** — `/users` and `/users/` are different routes.
- Wildcard matches **do not populate `request.params`**. Parse IDs from `request.path` yourself.
- In the **Portal UI**, enter the route pattern **without a leading `/`**. The incoming request still arrives with a leading `/`.
- **Overlapping patterns are rejected** at configuration time.

### Single-Router Pattern (recommended for many endpoints)

For multiple endpoints under one base path, configure **one** On Request Function with a prefix pattern (e.g., `v1/tasks*`) and dispatch by `request.method` + `request.path` inside the handler:

```javascript
// Portal route: v1/tasks*    (no leading slash in the Portal)
// Incoming request.path:     /v1/tasks/123/comments  (leading slash present)

export default async (request, response) => {
  const path = String(request.path || '').split('?')[0];        // defensive normalization
  const segments = path.replace(/^\//, '').split('/');          // ['v1','tasks','123','comments']

  try {
    if (segments[1] !== 'tasks') {
      return response.send({ error: 'Not Found' }, 404);
    }

    // /v1/tasks
    if (segments.length === 2 && request.method === 'GET') {
      return response.send({ items: [/* ... */] }, 200);
    }

    // /v1/tasks/{id}
    if (segments.length === 3) {
      const taskId = segments[2];
      if (request.method === 'GET')    return response.send({ id: taskId }, 200);
      if (request.method === 'PUT')    return response.send({ id: taskId, updated: true }, 200);
      if (request.method === 'DELETE') return response.send({ id: taskId, deleted: true }, 200);
    }

    // /v1/tasks/{id}/comments
    if (segments.length === 4 && segments[3] === 'comments') {
      const taskId = segments[2];
      if (request.method === 'GET')  return response.send({ taskId, comments: [] }, 200);
      if (request.method === 'POST') return response.send({ taskId, posted: true }, 201);
    }

    return response.send({ error: 'Not Found' }, 404);
  } catch (err) {
    console.error('Routing error:', err);
    return response.send({ error: 'Server error' }, 500);
  }
};
```

This avoids the "overlapping patterns rejected" problem when you have several endpoints under the same prefix.

### Why `request.params` is often empty

For On Request routes that use `*` wildcards, the runtime does not auto-extract named parameters. There is no `:userId` syntax. Anything you want to read out of the path must be parsed from `request.path` (which is the **raw** path, with leading `/`).

## Async/Await Best Practices

### Always Use async/await

```javascript
// CORRECT: async/await with try/catch
export default async (request) => {
  const db = require('kvstore');

  try {
    const data = await db.get('myKey');
    const count = await db.incrCounter('visits');

    request.message.data = data;
    request.message.visitCount = count;

    return request.ok();
  } catch (error) {
    console.error('Error:', error);
    return request.abort();
  }
};
```

```javascript
// BROKEN: missing `return` — handler resolves before the promise chain runs
export default (request) => {
  const db = require('kvstore');
  db.get('myKey')                              // promise dangles
    .then((data) => request.ok());
  // handler returns undefined here; runtime never sees ok()
};
```

### Promise Chains Are Acceptable If You Return Them

`async/await` is the recommended style. Promise chains remain valid for legacy or generated code, **provided you return the chain** so the runtime can await it:

```javascript
// Acceptable: chain is returned from the handler
export default (request) => {
  const db = require('kvstore');
  return db.get('myKey')
    .then((data) => {
      request.message.data = data;
      return request.ok();
    })
    .catch((err) => {
      console.error('Error:', err);
      return request.abort();
    });
};
```

### Parallel Operations with Promise.all

```javascript
export default async (request) => {
  const db = require('kvstore');
  const xhr = require('xhr');
  const vault = require('vault');

  try {
    // Execute independent operations in parallel
    const [userData, apiKey, config] = await Promise.all([
      db.get(`user:${request.message.userId}`),
      vault.get('external_api_key'),
      xhr.fetch('https://api.config.example.com/settings')
    ]);

    // Process results
    if (!userData) throw new Error('User not found');

    request.message.user = userData;
    request.message.config = JSON.parse(config.body);

    return request.ok();
  } catch (error) {
    console.error('Error:', error);
    return request.ok(); // Still allow message with error context
  }
};
```

## Execution Limits

| Limit | Value |
|-------|-------|
| Function chain depth | 3 executions (5 consecutive Functions total) |
| External calls per execution | 3 (XHR + PubNub API invocations) |
| Vault reads per execution | 10 (`vault.get(...)` calls) |
| Execution timeout | Few seconds |
| Response payload | Reasonable size |

> The 3-call XHR/PubNub cap and the 10-call vault cap are **configurable on request via PubNub Support** for Functions that legitimately need more. The chain depth and execution timeout are platform limits and not user-tunable.
>
> **`kvstore`, `crypto`, `jwt`, `uuid`, `utils`, `advanced_math`, `jsonpath`, and `codec/*` calls do NOT count** toward the 3-call cap — they are local-runtime helpers. See [Quirk 2: 3-Call Cap on XHR + PubNub API Calls](db-triggers-and-runtime-quirks.md) for the canonical accounting.

### Handling Limits

```javascript
export default async (request) => {
  const db     = require('kvstore');
  const pubnub = require('pubnub');
  const xhr    = require('xhr');

  try {
    // FREE — KVStore ops do NOT count toward the 3-call external cap
    const cached = await db.get('cachedConfig');
    await db.set('lastSeen', Date.now(), 60);

    // External call 1: HTTP fetch
    const resp = await xhr.fetch('https://example.com/enrich');

    // External call 2: publish
    await pubnub.publish({
      channel: 'enriched',
      message: { ...request.message, enriched: true }
    });

    // External call 3: fire analytics event (same 1-call cost as publish)
    await pubnub.fire({
      channel: 'analytics',
      message: { event: 'processed' }
    });

    // LIMIT REACHED — a 4th xhr.fetch / pubnub.* call would raise
    // "execution calls exceeds" and abort the handler.

    return request.ok();
  } catch (error) {
    console.error('Error:', error);
    return request.abort();
  }
};
```

KV reads/writes inside the same handler are free of this cap; only `xhr.fetch(...)` and `pubnub.*(...)` count. See [Quirk 2](db-triggers-and-runtime-quirks.md) for the canonical accounting.

## Logging and Debugging

```javascript
export default async (request) => {
  // Logs appear in PubNub Admin Portal
  console.log('Processing message:', JSON.stringify(request.message));
  console.log('Channel:', request.channels[0]);

  try {
    // Your logic
    console.log('Processing successful');
    return request.ok();
  } catch (error) {
    console.error('Processing failed:', error.message);
    console.error('Stack:', error.stack);
    return request.abort();
  }
};
```

View logs in the PubNub Admin Portal under Functions > Your Function > Logs.

## Function Template

```javascript
export default async (request) => {
  // 1. Import required modules
  const db = require('kvstore');
  const xhr = require('xhr');
  const vault = require('vault');
  const pubnub = require('pubnub');

  try {
    // 2. Validate input
    if (!request.message || !request.message.userId) {
      console.error('Invalid message format');
      return request.abort();
    }

    // 3. Perform business logic
    const userId = request.message.userId;
    const userData = await db.get(`user:${userId}`);

    // 4. Transform or enrich message
    request.message.enriched = true;
    request.message.userData = userData;
    request.message.processedAt = new Date().toISOString();

    // 5. Optional: side effects (analytics, notifications)
    await pubnub.fire({
      channel: 'analytics',
      message: { event: 'message_processed', userId }
    });

    // 6. Return success
    return request.ok();

  } catch (error) {
    // 7. Handle errors gracefully
    console.error('Function error:', error);

    // Option A: Block the message
    return request.abort();

    // Option B: Allow message with error flag
    // request.message.error = error.message;
    // return request.ok();
  }
};
```
