<!-- canonical-for: APPS_AND_KEYSETS -->
<!-- canonical-for: ENVIRONMENT_SEPARATION -->
<!-- used-by: pubnub-choose-docs-path, pubnub-security, pubnub-observability -->

# Apps, Keysets, and Environment Separation

The canonical reference for the PubNub app/keyset hierarchy and how to lay out dev/staging/prod isolation.

## App vs Keyset

A PubNub **App** is an organizational container in the Admin Portal. It usually maps 1:1 to a product (e.g., "Acme Chat", "Acme IoT"). An App holds one or more **Keysets**.

A **Keyset** is the unit of isolation: it has its own publish key, subscribe key, secret key, and feature configuration. Two clients with different keysets can never see each other's traffic.

```
App: Acme Chat
├── Keyset: dev
│   ├── pub-c-dev-...
│   ├── sub-c-dev-...
│   ├── sec-c-dev-...   (server only)
│   └── add-ons: Presence + Persistence + Access Manager
├── Keyset: staging
│   └── (same shape, different keys)
└── Keyset: production
    └── (same shape, different keys, hardened settings)
```

## The Three Key Types

### Publish Key (`pub-c-...`)

- Sent from clients and servers to publish messages.
- Required for [`publish()`, `signal()`, `fire()`, and `pubnub.publish()`](../../pubnub-app-developer/references/publish-subscribe.md) from a Function.
- Safe to ship in client code (presence in source code is expected).
- Subject to Access Manager when enabled — even with the publish key, a client without a valid token cannot publish to restricted channels.

### Subscribe Key (`sub-c-...`)

- Sent from clients and servers to receive messages.
- Required for `subscribe()`, [`hereNow()`](../../pubnub-presence/references/presence-events.md), [`fetchMessages()`](../../pubnub-history/references/pagination-and-ordering.md), and most read operations.
- Safe to ship in client code.
- Also required to instantiate any PubNub SDK.

### Secret Key (`sec-c-...`)

- **Server-side only.** Never appears in any client code, mobile app bundle, browser bundle, or public repository.
- Required to grant Access Manager tokens (`grantToken()`) and to perform certain admin operations.
- A leaked secret key is a security incident: rotate immediately.
- Some operations (like Functions deployment) use the secret key indirectly through the Admin API; that is still server-side use.

## Environment Separation Strategy

### One Keyset Per Environment

Use distinct keysets for **dev**, **staging**, and **production**. This is the single most important rule in this document.

Why:

- A test publisher in dev cannot accidentally reach production subscribers.
- You can enable verbose logging or shorter token TTLs in dev without affecting production.
- You can rate-limit dev separately so a runaway test loop doesn't burn through your production transaction budget.
- You can rotate or revoke a non-production keyset without coordinating with production traffic.

### Naming Convention

Pick a convention and apply it everywhere — Admin Portal labels, env var names, secrets-manager paths, alerting tags:

```text
ACME_CHAT_DEV_PUBLISH_KEY
ACME_CHAT_DEV_SUBSCRIBE_KEY
ACME_CHAT_DEV_SECRET_KEY
ACME_CHAT_STAGING_PUBLISH_KEY
ACME_CHAT_STAGING_SUBSCRIBE_KEY
ACME_CHAT_STAGING_SECRET_KEY
ACME_CHAT_PROD_PUBLISH_KEY
ACME_CHAT_PROD_SUBSCRIBE_KEY
ACME_CHAT_PROD_SECRET_KEY
```

### Per-Environment Add-on Configuration

Enable different add-ons in different environments based on need. Common pattern:

| Add-on | dev | staging | prod |
|---|---|---|---|
| Presence | on (short heartbeat for fast iteration) | on (prod settings) | on (prod settings) |
| [Message Persistence](../../pubnub-history/references/pagination-and-ordering.md) | on (1 day retention) | on (matches prod) | on (per product retention) |
| Access Manager | on | on (mirrors prod TTLs) | on (production TTLs) |
| [Stream Controller](../../pubnub-scale/references/scaling-patterns.md) | on | on | on |
| Functions | on (in lower-cost mode if available) | on | on |
| Events & Actions | off (for cost) | on (mirrors prod) | on |

Drift between staging and prod add-on configuration is a common cause of "works in staging, fails in prod" incidents — pick a configuration parity rule and document it.

## Loading Keys at Runtime

### Server (Node.js example)

```javascript
const PubNub = require('pubnub');

function getEnvOrThrow(name) {
  const value = process.env[name];
  if (!value) {
    throw new Error(`Missing required env var: ${name}`);
  }
  return value;
}

const env = process.env.NODE_ENV || 'dev';
const prefix = `ACME_CHAT_${env.toUpperCase()}`;

const pubnub = new PubNub({
  publishKey:   getEnvOrThrow(`${prefix}_PUBLISH_KEY`),
  subscribeKey: getEnvOrThrow(`${prefix}_SUBSCRIBE_KEY`),
  secretKey:    getEnvOrThrow(`${prefix}_SECRET_KEY`),
  userId: 'server-' + require('os').hostname(),
});
```

### Client

Clients never see the secret key. Inject only publish + subscribe keys at build time, scoped to environment:

```javascript
const env = process.env.REACT_APP_ENV;
export const pubnubConfig = {
  publishKey:   process.env.REACT_APP_PN_PUBLISH_KEY,
  subscribeKey: process.env.REACT_APP_PN_SUBSCRIBE_KEY,
  userId:       getCurrentUserId(),
};
```

For the actual client SDK initialization patterns (listeners, error handling, userId requirements), see [pubnub-app-developer/references/sdk-patterns.md](../../pubnub-app-developer/references/sdk-patterns.md).

## Common Anti-Patterns

| Anti-pattern | Why it fails | Fix |
|---|---|---|
| Single keyset shared across dev + prod | Test traffic reaches real users; cost mixing | One keyset per env |
| Secret key in mobile app bundle | Anyone can extract it from the binary | Server-only secret key |
| Hardcoded keys in source files | Leaks on first repo clone or CI log | Env vars + secrets manager |
| Same keyset between two unrelated apps | Accidental subscribe to other app's channels | One App per product |
| Reusing dev keys when shipping the demo to prod | Production runs with debug-tier limits | Separate prod keyset |
| Rotating secret key without re-issuing tokens | All in-flight tokens immediately invalidated | Coordinate rotation, see [key-rotation-and-hygiene.md](key-rotation-and-hygiene.md) |

## MCP Tools

- **`manage_apps`** — list, create, inspect, update apps
- **`manage_keysets`** — list, create, inspect, update keysets, toggle add-ons

## Related Reading

- [key-rotation-and-hygiene.md](key-rotation-and-hygiene.md) — rotation cadence and procedure
- [demo-keys.md](demo-keys.md) — when demo keys are acceptable
- [custom-origin.md](custom-origin.md) — vanity domain setup
- [pubnub-security/references/access-manager.md](../../pubnub-security/references/access-manager.md) — using the secret key to grant tokens
