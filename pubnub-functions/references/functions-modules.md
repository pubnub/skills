<!-- canonical-for: FUNCTION_MODULES -->
<!-- used-by: -->

> **Cross-references:** [Vault for secrets / secret-key handling](../../pubnub-keyset-management/references/key-rotation-and-hygiene.md). [Crypto module is a Function-side option for the same end-to-end message encryption](../../pubnub-security/references/encryption.md). The PubNub module's [`pubnub.publish(`](../../pubnub-app-developer/references/publish-subscribe.md) call from inside a Function uses the same [SDK userId/UUID semantics](../../pubnub-app-developer/references/sdk-patterns.md) and is subject to the [3-chain rule](functions-chaining.md).

# PubNub Functions 2.0 Modules

## Available Modules

| Module | Purpose |
|--------|---------|
| `kvstore` | Persistent key-value storage |
| `xhr` | HTTP requests to external APIs |
| `vault` | Secure storage for secrets |
| `pubnub` | Publish, signal, fire messages |
| `crypto` | Cryptographic operations |
| `utils` | Utility functions |
| `uuid` | UUID generation |
| `jwt` | JWT creation and verification |
| `codec/auth` | PubNub Access Manager auth helpers |
| `codec/base64` | Base64 encoding/decoding |
| `codec/query_string` | URL query string parsing |
| `advanced_math` | Geospatial and math helpers |
| `jsonpath` | JSONPath querying for nested JSON |
| `ugc` | User-generated content moderation helpers |

## KVStore Module

Persistent key-value storage across function executions.

### Basic Operations

```javascript
const db = require('kvstore');

// Set with TTL (minutes)
await db.set('user:123', { name: 'Alice', email: 'alice@example.com' }, 1440);  // 24 hours

// Get value
const user = await db.get('user:123');

// Remove value
await db.removeItem('user:123');

// Get all keys
const keys = await db.getKeys();
```

### Atomic Counters

```javascript
const db = require('kvstore');

// Increment counter (atomic)
const newCount = await db.incrCounter('page_views');
console.log('Total views:', newCount);

// Get counter value
const count = await db.getCounter('page_views');

// Counters are separate from regular keys
// Use incrCounter/getCounter, not set/get
```

### Example: Rate Limiter

```javascript
const db = require('kvstore');

async function checkRateLimit(userId, limit, windowMinutes) {
  const key = `rate:${userId}:${Math.floor(Date.now() / (windowMinutes * 60 * 1000))}`;
  const count = await db.incrCounter(key);

  if (count === 1) {
    // First request in window - set TTL
    await db.set(`${key}_ttl`, true, windowMinutes * 2);
  }

  return count <= limit;
}

export default async (request) => {
  const allowed = await checkRateLimit(request.message.userId, 100, 60);

  if (!allowed) {
    request.message.rateLimited = true;
    return request.abort();
  }

  return request.ok();
};
```

## XHR Module

Make HTTP requests to external APIs.

### GET Request

```javascript
const xhr = require('xhr');

const response = await xhr.fetch('https://api.example.com/data');
const data = JSON.parse(response.body);
console.log('Status:', response.status);
```

### POST Request

```javascript
const xhr = require('xhr');

const response = await xhr.fetch('https://api.example.com/webhook', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': 'Bearer your-token'
  },
  body: JSON.stringify({
    event: 'message_received',
    data: request.message
  })
});

if (response.status === 200) {
  console.log('Webhook delivered');
}
```

### Example: Slack Notification

```javascript
export default async (request) => {
  const xhr = require('xhr');
  const vault = require('vault');

  try {
    const webhookUrl = await vault.get('slack_webhook_url');

    await xhr.fetch(webhookUrl, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        text: `New alert: ${request.message.alert}`,
        channel: '#alerts'
      })
    });

    return request.ok();
  } catch (error) {
    console.error('Slack notification failed:', error);
    return request.ok();  // Don't block the original message
  }
};
```

## Vault Module

Secure storage for API keys and secrets.

> **Execution limit:** A single Function execution can perform up to **10 vault lookups** (`vault.get('KEY')`) before failing. Load only the keys you need per invocation, and use a single consistent key casing (e.g., uppercase) to avoid accidental duplicate lookups. Vault reads do **not** count toward the 3-call external-operation cap (vault is local to the runtime).

```javascript
const vault = require('vault');

// Get secret (set via PubNub Admin Portal)
const apiKey = await vault.get('my_api_key');

if (!apiKey) {
  throw new Error('API key not configured in vault');
}

// Use the secret
const response = await xhr.fetch('https://api.example.com/data', {
  headers: { 'Authorization': `Bearer ${apiKey}` }
});
```

**Setting Vault Secrets:**
1. Go to PubNub Admin Portal
2. Navigate to Functions > Your Module
3. Click "My Secrets"
4. Add key-value pairs

## PubNub Module

Publish messages, signals, and fire events.

```javascript
const pubnub = require('pubnub');

// Publish message (delivered to subscribers, triggers functions)
await pubnub.publish({
  channel: 'notifications',
  message: { type: 'alert', text: 'New message received' }
});

// Signal (lightweight, no persistence)
await pubnub.signal({
  channel: 'typing',
  message: { user: 'alice', typing: true }
});

// Fire (triggers functions only, not subscribers)
await pubnub.fire({
  channel: 'analytics',
  message: { event: 'page_view', timestamp: Date.now() }
});
```

### publish vs fire vs signal

| Method | Delivers to Subscribers | Triggers Functions | Stored in History |
|--------|------------------------|-------------------|-------------------|
| `publish` | Yes | Yes | Yes (if enabled) |
| `fire` | No | Yes | No |
| `signal` | Yes | Yes | No |

## Crypto Module

Cryptographic operations.

```javascript
const crypto = require('crypto');

// HMAC signature
const signature = await crypto.hmac(
  'secretKey',
  'data to sign',
  crypto.ALGORITHM.HMAC_SHA256
);

// Hash
const hash = await crypto.sha256('hello world');

// MD5 (for checksums, not security)
const checksum = await crypto.md5('file contents');
```

## UUID Module

Generate and validate UUIDs.

```javascript
const { v4, validate } = require('uuid');

// Generate random UUID
const id = v4();  // e.g., '9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d'

// Validate UUID
if (validate(id)) {
  console.log('Valid UUID');
}
```

## JWT Module

Create and verify JSON Web Tokens.

```javascript
const jwt = require('jwt');
const vault = require('vault');

// Create token
const secret = await vault.get('jwt_secret');
const token = jwt.sign({ userId: '123', role: 'admin' }, secret, {
  expiresIn: '1h'
});

// Verify token
try {
  const decoded = jwt.verify(token, secret);
  console.log('User ID:', decoded.userId);
} catch (error) {
  console.error('Invalid token:', error);
}
```

## Codec Modules

### Base64

```javascript
const base64 = require('codec/base64');

// Encode
const encoded = base64.btoa('Hello World');

// Decode
const decoded = base64.atob(encoded);
```

### Query String

```javascript
const qs = require('codec/query_string');

// Parse query string
const params = qs.parse('name=Alice&age=30');
// { name: 'Alice', age: '30' }

// Stringify object
const queryString = qs.stringify({ name: 'Bob', age: 25 });
// 'name=Bob&age=25'
```

### Auth

The `codec/auth` codec exposes PubNub Access Manager auth helpers (signature building, token formats). It is rarely used directly from user code — most authorization is handled via the `pubnub` and `vault` modules. Consult the PubNub Functions documentation for the specific helpers exposed by your runtime version.

```javascript
const auth = require('codec/auth');
// Use auth.<helper>(...) — refer to the runtime's documented API.
```

## Utils Module

```javascript
const utils = require('utils');

// Generate random number
const random = utils.random();  // 0-1

// Check if numeric
if (utils.isNumeric('123')) {
  console.log('Is a number');
}
```

## Advanced Math Module

PubNub Functions ship an `advanced_math` module for math and geospatial helpers (typical operations: rounding, distance, normalization). The exact surface depends on the Functions runtime version; consult the PubNub Functions documentation for the current `advanced_math` API.

```javascript
const math = require('advanced_math');

// Use math.<helper>(...) — refer to the runtime's documented API.
// Use this module instead of native `Math.*` when you need server-side
// behavior consistent with other PubNub computations.
```

## JSONPath Module

Query nested JSON with [JSONPath](https://goessner.net/articles/JsonPath/) expressions. Useful for extracting fields from deeply nested message payloads or webhook bodies without writing brittle optional-chaining ladders.

```javascript
const jp = require('jsonpath');

const message = {
  user: { id: 'u-123', tier: 'pro' },
  items: [
    { sku: 'A', qty: 2 },
    { sku: 'B', qty: 5 }
  ]
};

// Pull a single field
const tier = jp.query(message, '$.user.tier')[0];     // 'pro'

// Pull every sku from items
const skus = jp.query(message, '$.items[*].sku');     // ['A', 'B']
```

JSONPath is convenient when message shapes vary slightly across producers — you can query optional fields without throwing on `undefined`.

## UGC Module

User-Generated Content moderation helpers. Typical use is calling a lightweight moderation check from a Before Publish Function on chat or comment channels, and aborting on toxic content.

```javascript
const ugc = require('ugc');

// The specific surface depends on the Functions runtime version.
// Consult the PubNub Functions documentation for the current `ugc` API.
```

For full-featured moderation (toxicity scoring, multi-category classification, threshold tuning), call an external moderation API via `xhr` (see [Pattern 3: Content Moderation](functions-patterns.md)). Use the built-in `ugc` module for lightweight checks that don't require an external API call — they don't consume an entry from the 3-call external-operation cap.

## Module Usage Summary

```javascript
export default async (request) => {
  // Import at top of function
  const db = require('kvstore');
  const xhr = require('xhr');
  const vault = require('vault');
  const pubnub = require('pubnub');
  const crypto = require('crypto');
  const { v4: uuidv4 } = require('uuid');

  try {
    // Get secrets from vault
    const apiKey = await vault.get('external_api_key');

    // Store data in KV store
    await db.set('lastProcessed', Date.now(), 60);

    // Make external API call
    const response = await xhr.fetch('https://api.example.com/data', {
      headers: { 'Authorization': `Bearer ${apiKey}` }
    });

    // Generate unique ID
    const messageId = uuidv4();

    // Fire analytics event
    await pubnub.fire({
      channel: 'analytics',
      message: { messageId, processed: true }
    });

    return request.ok();
  } catch (error) {
    console.error('Error:', error);
    return request.abort();
  }
};
```
