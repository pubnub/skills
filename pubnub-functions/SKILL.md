---
name: pubnub-functions
description: Create, configure, and deploy PubNub Functions 2.0 event handlers, triggers, and serverless endpoints. Covers Before/After Publish, On Request, On Interval; built-in modules (kvstore, xhr, vault, pubnub, crypto, jwt, ugc, jsonpath, advanced_math, codec/*); chaining (3 hops, 5 consecutive, Chaining vs Forking, kvstore state sharing); runtime quirks (3-call external cap, 10-call vault cap, cold start, request.path normalization, vault availability, sendFile message); DB-trigger patterns; and bundling/TypeScript workflow (esbuild externals, 64KB guard, __require shim stripping, default-export shape). Use when building real-time message transformations, edge data processing, REST endpoints backed by PubNub, webhook integrations, or shipping bundled/transpiled TypeScript Functions from inside the message pipeline.
license: PubNub
metadata:
  author: pubnub
  version: "0.3.0"
  domain: real-time
  triggers: pubnub, pubnub functions, functions, serverless, edge, kvstore, webhook, transform, event handler, real-time functions, message processing, before publish, after publish, on request, on interval, function chaining, function bundling, typescript, esbuild, vault, request path, sendFile, ugc, jsonpath, advanced_math, on request routing
  role: specialist
  scope: implementation
  output-format: code
---

# PubNub Functions 2.0

You are the PubNub Functions specialist. Your role is to help developers build serverless event handlers and HTTP endpoints inside the PubNub message pipeline.

## When to Use This Skill

Invoke this skill when:
- Transforming messages in flight (Before Publish)
- Triggering side effects after delivery (After Publish)
- Building HTTP endpoints backed by PubNub state (On Request)
- Running scheduled work (On Interval)
- Sync-mirroring messages to external systems (DB triggers)
- Working with PubNub modules: KVStore, XHR, Vault, PubNub, Crypto, JWT, UUID

## Core Workflow

1. **Identify Function Type**: Before Publish, After Publish, On Request, or On Interval.
2. **Design Logic**: Plan the transformation, integration, or business logic.
3. **Implement Function**: Write async/await code with proper error handling.
4. **Use Modules**: Leverage kvstore, xhr, vault, pubnub, crypto modules.
5. **Validate Implementation**: Verify no hardcoded secrets (use vault); confirm every code path returns the trigger's completion call (`request.ok()` / `request.abort()` / `response.send()`); check that `xhr` + `pubnub` external calls stay within the **3-call cap** (KVStore, vault, and pure-CPU helpers do **not** count — see [Quirk 2](references/db-triggers-and-runtime-quirks.md)); ensure proper try/catch wrapping.
6. **Handle Response**: Return `request.ok()` / `request.abort()` or `response.send()` appropriately.
7. **Configure Channel Patterns**: Wildcard patterns must end with `.*`, max two literal segments before wildcard.
8. **Test in Staging**: Test in PubNub Admin Portal with sample messages before enabling on production channels.
9. **Deploy to Production**: Enable on live channel patterns and monitor logs.

## Reference Guide

| Reference | Purpose |
|-----------|---------|
| [functions-basics.md](references/functions-basics.md) | Function structure, event types, channel-pattern + On Request URI routing, async/await + Promise chains, execution limits |
| [functions-modules.md](references/functions-modules.md) | All built-in modules: KVStore, XHR, Vault (with 10-call cap), PubNub, Crypto, JWT, UUID, JSONPath, Advanced Math, UGC, Codec |
| [functions-patterns.md](references/functions-patterns.md) | Common patterns: counters, transforms, moderation, webhooks, REST endpoints, rate limiting, auth middleware |
| [functions-chaining.md](references/functions-chaining.md) | 3-hop rule, 5-consecutive Functions cap, Chaining vs Forking, kvstore state sharing, channel namespace hygiene |
| [db-triggers-and-runtime-quirks.md](references/db-triggers-and-runtime-quirks.md) | DB-mirror patterns; 11 runtime quirks (cold start, 3-call cap accounting, handler-scoped require, request.path normalization, vault availability, sendFile message) |
| [bundling-and-typescript.md](references/bundling-and-typescript.md) | TypeScript + esbuild workflow: externals list, 64 KB size guard, `__require` shim stripping, default-export shape, require placement, minification gotchas, LLM do/don't checklist |

## Key Implementation Requirements

> **Cross-references:** Built on [pub/sub basics](../pubnub-app-developer/references/publish-subscribe.md). Use [vault for secrets — including the secret key](../pubnub-keyset-management/references/keysets-and-environments.md) and the [key rotation guide](../pubnub-keyset-management/references/key-rotation-and-hygiene.md). For [logging correlation in functions](../pubnub-observability/references/logging-correlation.md) include the four required fields. [Wildcard pattern subscribe is owned by pubnub-scale](../pubnub-scale/references/scaling-patterns.md).

### Function Structure

```javascript
export default async (request) => {
  const db = require('kvstore');
  const xhr = require('xhr');

  try {
    return request.ok();
  } catch (error) {
    console.error('Error:', error);
    return request.abort();
  }
};
```

### HTTP Endpoint Function

```javascript
export default async (request, response) => {
  try {
    const body = await request.json();
    return response.send({ success: true }, 200);
  } catch (error) {
    return response.send({ error: 'Server error' }, 500);
  }
};
```

## Constraints

- **Function chaining**: maximum **3 chained executions per inbound publish**; up to **5 consecutive Functions** in a single sequence ([functions-chaining.md](references/functions-chaining.md)).
- **External calls per execution**: maximum **3** (`xhr.fetch` + `pubnub` API invocations). KVStore, vault, and pure-CPU helpers (`crypto`, `jwt`, `uuid`, `utils`, `advanced_math`, `jsonpath`, `codec/*`) do **not** count ([Quirk 2](references/db-triggers-and-runtime-quirks.md)).
- **Vault reads per execution**: maximum **10** `vault.get(...)` calls. Vault has its own per-execution ceiling separate from the 3-call external cap ([Vault Module](references/functions-modules.md), [Quirk 10](references/db-triggers-and-runtime-quirks.md)).
- The 3-call external cap and 10-call vault cap are **configurable on request via PubNub Support**.
- Prefer `async`/`await`; Promise chains are acceptable when **returned** from the handler.
- Always wrap logic in `try`/`catch` and ensure every code path returns the trigger's completion call (`request.ok()` / `request.abort()` / `response.send()`).
- Use `vault` for secrets, never hardcode. Guard `vault.get(...)` against the module being unavailable ([Quirk 10](references/db-triggers-and-runtime-quirks.md)).
- **Channel-trigger wildcards** must end with `.*`, max two literal segments before the wildcard (Before/After Publish). **On Request URI routing** uses different rules — see [functions-basics.md](references/functions-basics.md).
- **Bundling/transpiling**: keep all Functions built-ins externalized, default-export as the first statement, `require()` calls inside the handler, and no `__require` / `Dynamic require of` shims in the final bundle. See [bundling-and-typescript.md](references/bundling-and-typescript.md).
- Cold start can add latency; structure logic to be idempotent so retries are safe (see [idempotent publish](../pubnub-reliability/references/idempotent-publish.md)).

## MCP Tools

- **`create_pubnub_function`** — scaffold a Function from this skill's templates
- **`get_sdk_documentation`** — pull current Function module API references (see [intent-to-tool routing](../pubnub-choose-docs-path/references/intent-to-tool.md))

## See Also

- **pubnub-app-developer** — for [pub/sub semantics, listeners](../pubnub-app-developer/references/publish-subscribe.md), [SDK patterns](../pubnub-app-developer/references/sdk-patterns.md)
- **pubnub-events-and-actions** — alternative for **routing** (no transformation); see [Events & Actions overview](../pubnub-events-and-actions/SKILL.md)
- **pubnub-illuminate** — alternative for **threshold-triggered** automation (no per-message transform); see [Decisions](../pubnub-illuminate/references/decisions-4-step-workflow.md)
- **pubnub-app-context** — Functions can read/write [App Context users/channels](../pubnub-app-context/references/users.md)
- **pubnub-security** — Functions sign with the [secret key from Vault](../pubnub-security/references/access-manager.md)
- **pubnub-reliability** — server-side [idempotency check via KVStore](../pubnub-reliability/references/idempotent-publish.md)
- **pubnub-observability** — [function execution metrics](../pubnub-observability/references/usage-metrics.md), [incident triage](../pubnub-observability/references/incident-runbook.md) for slow Functions
- **pubnub-choose-docs-path** — for routing other PubNub questions

## Output Format

When providing implementations:
1. Include complete, working function code.
2. Show proper async/await with try/catch.
3. Explain module usage and imports.
4. Note channel pattern configuration.
5. Include deployment instructions.
6. State the function type at the top of every snippet.
