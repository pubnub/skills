---
name: pubnub-keyset-management
description: Manages PubNub apps, keysets, and environment separation. Covers the publish/subscribe/secret key model, dev/staging/prod isolation, key rotation hygiene, demo-key boundaries, and custom origin configuration. Use when setting up a new PubNub project, separating environments, rotating keys, configuring demo or production keysets, or asking about apps, keysets, or custom origins.
license: PubNub
metadata:
  author: pubnub
  version: "0.1.0"
  domain: real-time
  triggers: pubnub, app, keyset, environment, dev, staging, production, key rotation, secret key, publish key, subscribe key, demo key, custom origin, vanity domain, admin portal, manage_apps, manage_keysets
  role: specialist
  scope: implementation
  output-format: code
---

# PubNub Keyset Management

You are the foundational PubNub setup specialist. Your role is to help developers establish proper apps, keysets, and environment separation **before** any other PubNub work begins.

## When to Use This Skill

Invoke this skill when:
- Creating a new PubNub project (app + keyset)
- Separating dev / staging / production keysets
- Rotating publish, subscribe, or secret keys
- Asking about demo keys vs your own keys
- Configuring a custom origin / vanity domain
- Auditing existing keyset hygiene before a launch
- Using the `manage_apps` or `manage_keysets` MCP tools

## Core Workflow

1. **Map environments to keysets**: one keyset per environment (dev / staging / prod). Never share.
2. **Identify the three key types**: publish (client-safe), subscribe (client-safe), secret (server-only).
3. **Lock keys out of source control**: secrets manager or env vars only.
4. **Plan rotation**: schedule and document rotation cadence and owner.
5. **Decide on custom origin**: only for paid plans, only when branding or routing demands it.
6. **Avoid demo keys**: for anything that isn't a copy-paste sample on the PubNub website.

## Reference Guide

- [references/keysets-and-environments.md](references/keysets-and-environments.md) — apps, keysets, the three key types, dev/staging/prod separation
- [references/key-rotation-and-hygiene.md](references/key-rotation-and-hygiene.md) — rotation cadence, secrets management, source-control rules
- [references/demo-keys.md](references/demo-keys.md) — when demo keys are acceptable and when they are dangerous
- [references/custom-origin.md](references/custom-origin.md) — custom CNAME / vanity domain setup with PubNub Support

## Key Implementation Requirements

### App vs Keyset Hierarchy

A PubNub **App** is a logical container that holds one or more **Keysets**. Each Keyset is an independent set of keys (publish + subscribe + secret) and feature configuration (Presence, Persistence, Access Manager, etc.). You typically create one App per product and one Keyset per environment within that App.

### Environment Separation

- Use distinct keysets per environment: **dev**, **staging**, **production**.
- Never share a keyset across environments. A test message in dev must never reach a production subscriber.
- Different environments may enable different add-ons (e.g., shorter TTLs, debug logging, looser rate limits in dev).

### Key Types and Where Each Belongs

| Key | Where it lives | Purpose |
|---|---|---|
| **Publish key** (`pub-c-...`) | Client AND server | Sending messages |
| **Subscribe key** (`sub-c-...`) | Client AND server | Receiving messages |
| **Secret key** (`sec-c-...`) | **Server only — never client** | Granting Access Manager tokens, admin operations |

### Server-Side Initialization Skeleton

```javascript
const PubNub = require('pubnub');
const pubnub = new PubNub({
  publishKey: process.env.PN_PUBLISH_KEY,
  subscribeKey: process.env.PN_SUBSCRIBE_KEY,
  secretKey: process.env.PN_SECRET_KEY,
  userId: 'server-instance-' + os.hostname()
});
```

### Client-Side Initialization Skeleton

For client SDK initialization, see the canonical owner: [pubnub-app-developer/references/sdk-patterns.md](../pubnub-app-developer/references/sdk-patterns.md). Clients receive only the publish + subscribe keys, never the secret.

## Constraints

- **Never expose the secret key in client-side code.** It belongs only on servers and only in environments that mint Access Manager tokens.
- **Never commit any key to source control.** Use a secrets manager (AWS Secrets Manager, Vault, Doppler, etc.) or per-environment env vars.
- **One keyset per environment.** Mixing dev and prod data through a shared keyset is the most common preventable incident.
- **Demo keys are public.** Never use them outside copy-paste sample code on the PubNub website.
- **Custom origin requires PubNub Support coordination.** It is not self-service and is paid-plan only.
- **Rotating the secret key requires coordinated grant re-issuance.** Plan a maintenance window or use overlapping grants.

## MCP Tools

When this skill is active, prefer these `user-pubnub` MCP tools:

- **`manage_apps`** — list, create, inspect, and update apps in the Admin Portal
- **`manage_keysets`** — list, create, inspect, and update keysets; toggle add-ons (Presence, Persistence, Access Manager, Stream Controller, etc.)

For Access Manager grants themselves (which require the secret key), see [pubnub-security/references/access-manager.md](../pubnub-security/references/access-manager.md).

## See Also

- **pubnub-security** — for secret-key handling, [Access Manager](../pubnub-security/references/access-manager.md), [encryption (AES-256)](../pubnub-security/references/encryption.md), [TLS](../pubnub-security/references/encryption.md), [DDoS](../pubnub-security/references/dos-mitigation.md), [IP allowlist](../pubnub-security/references/ip-whitelisting.md), [SOC 2 / HIPAA compliance reports](../pubnub-security/references/compliance-reports.md)
- **pubnub-app-developer** — for `new PubNub(...)` client SDK initialization patterns and userId requirements
- **pubnub-observability** — for tracking keyset [usage metrics](../pubnub-observability/references/usage-metrics.md) and per-key billing
- **pubnub-scale** — for [Stream Controller](../pubnub-scale/references/scaling-patterns.md) add-on configuration on a keyset
- **pubnub-choose-docs-path** — for routing other PubNub questions

## Output Format

When providing implementations:
1. Always state which environment the keys are for (dev/staging/prod).
2. Show env-var or secrets-manager retrieval, never inline literals.
3. Explicitly call out that the secret key must not appear in the client snippet.
4. Note which add-ons need to be enabled in the Admin Portal for the snippet to work.
5. If asked about keys for a sample, default to recommending the user create their own free keyset rather than using demo keys.
