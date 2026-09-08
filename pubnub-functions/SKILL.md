---
name: pubnub-functions
description: Create, configure, and deploy PubNub Functions 2.0 event handlers, triggers, and serverless endpoints. Covers Before/After Publish, On Request, On Interval; built-in modules (kvstore, xhr, vault, pubnub, crypto, jwt, ugc, jsonpath, advanced_math, codec/*); chaining (depth caps, Chaining vs Forking, kvstore state sharing); runtime quirks (independent per-module execution budgets, cold start, request.path normalization, vault availability, sendFile message); DB-trigger patterns; and bundling/TypeScript workflow (esbuild externals, bundle size guard, __require shim stripping, default-export shape). Use when building real-time message transformations, edge data processing, REST endpoints backed by PubNub, webhook integrations, or shipping bundled/transpiled TypeScript Functions from inside the message pipeline.
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

> **Precedence:** PubNub MCP tools and pubnub.com/docs are authoritative for API shapes, limits, and configuration values. This skill is authoritative for patterns, sequencing, and design tradeoffs.



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
5. **Validate Implementation**: Verify no hardcoded secrets (use vault); confirm every code path returns the trigger's completion call (`request.ok()` / `request.abort()` / `response.send()`); check per-module execution budgets (XHR, KV Store, PubNub API, Vault are independent — see [Quirk 2](references/db-triggers-and-runtime-quirks.md)); ensure proper try/catch wrapping.
6. **Handle Response**: Return `request.ok()` / `request.abort()` or `response.send()` appropriately.
7. **Configure Channel Patterns**: Wildcard patterns must end with `.*`, max two literal segments before wildcard.
8. **Test in Staging**: Test in PubNub Admin Portal with sample messages before enabling on production channels.
9. **Deploy to Production**: Enable on live channel patterns and monitor logs.

## Reference Guide

| Reference | Purpose |
|-----------|---------|
| [functions-basics.md](references/functions-basics.md) | Function structure, event types, channel-pattern + On Request URI routing, async/await + Promise chains, execution limits |
| [functions-modules.md](references/functions-modules.md) | All built-in modules: KVStore, XHR, Vault, PubNub, Crypto, JWT, UUID, JSONPath, Advanced Math, UGC, Codec |
| [functions-patterns.md](references/functions-patterns.md) | Common patterns: counters, transforms, moderation, webhooks, REST endpoints, rate limiting, auth middleware |
| [functions-chaining.md](references/functions-chaining.md) | Chain depth caps, Chaining vs Forking, kvstore state sharing, channel namespace hygiene |
| [db-triggers-and-runtime-quirks.md](references/db-triggers-and-runtime-quirks.md) | DB-mirror patterns; 11 runtime quirks (cold start, per-module execution limits, handler-scoped require, request.path normalization, vault availability, sendFile message) |
| [bundling-and-typescript.md](references/bundling-and-typescript.md) | TypeScript + esbuild workflow: externals list, bundle size guard, `__require` shim stripping, default-export shape, require placement, minification gotchas, LLM do/don't checklist |

## Key Implementation Requirements


| Requirement | Rule |
|-------------|------|
| Handler shape | Default async export; every path returns `ok`/`abort`/`send` |
| Secrets | `vault.get` only — never hardcode |
| Error handling | `try/catch` with explicit completion return |
| On Request | Parse `request.path` yourself; wildcard routes do not fill `request.params` |
| Budgets | Count xhr / pubnub / kvstore / vault ops per execution ([Quirk 2](references/db-triggers-and-runtime-quirks.md)) |

## Constraints

- **Function chaining**: the platform enforces chain-depth and consecutive-Function caps per inbound publish — retrieve current values via **`how_to`** (`understand-pubnub-functions-limits-and-constraints`) before designing multi-hop pipelines ([functions-chaining.md](references/functions-chaining.md)).
- **Per-module execution limits**: XHR, KV Store, PubNub API, and Vault each have an **independent** per-execution budget — they do **not** share a combined pool ([Quirk 2](references/db-triggers-and-runtime-quirks.md)). Retrieve current default caps via **`how_to`** before counting ops in a handler. Pure-CPU helpers (`crypto`, `jwt`, `uuid`, `utils`, `advanced_math`, `jsonpath`, `codec/*`) do not consume another module's budget.
- Per-module caps are **configurable on request via PubNub Support** — confirm current ceilings with **`how_to`** or Functions docs.
- Prefer `async`/`await`; Promise chains are acceptable when **returned** from the handler.
- Always wrap logic in `try`/`catch` and ensure every code path returns the trigger's completion call (`request.ok()` / `request.abort()` / `response.send()`).
- Use `vault` for secrets, never hardcode. Guard `vault.get(...)` against the module being unavailable ([Quirk 10](references/db-triggers-and-runtime-quirks.md)).
- **Channel-trigger wildcards** must end with `.*`, max two literal segments before the wildcard (Before/After Publish). **On Request URI routing** uses different rules — see [functions-basics.md](references/functions-basics.md).
- **Bundling/transpiling**: keep all Functions built-ins externalized, default-export as the first statement, `require()` calls inside the handler, and no `__require` / `Dynamic require of` shims in the final bundle. See [bundling-and-typescript.md](references/bundling-and-typescript.md).
- Cold start can add latency; structure logic to be idempotent so retries are safe (see [idempotent publish](../pubnub-reliability/references/idempotent-publish.md)).

## MCP Tools

For **limits, chain depth, and runtime constraints**, call **`how_to`** (`understand-pubnub-functions-limits-and-constraints`) first — do not rely on web search or cached numbers.

- **`how_to`** — Functions execution limits, chain depth, timeout, and per-module budgets (authoritative for numeric caps)
- **`manage_functions`** (`resource=package`, `operation=create`) — create a Functions package/revision from this skill's templates
- **`get_sdk_documentation`** — pull current Function module API references (see [intent-to-tool routing](../pubnub-choose-docs-path/references/intent-to-tool.md))

## See Also

- **pubnub-app-developer** — for [pub/sub semantics, listeners](../pubnub-app-developer/SKILL.md), [SDK patterns](../pubnub-app-developer/SKILL.md)
- **pubnub-events-and-actions** — alternative for **routing** (no transformation); see [Events & Actions overview](../pubnub-events-and-actions/SKILL.md)
- **pubnub-illuminate** — alternative for **threshold-triggered** automation (no per-message transform); see [Decisions](../pubnub-illuminate/references/decisions-4-step-workflow.md)
- **pubnub-app-context** — Functions can read/write [App Context users/channels](../pubnub-app-context/references/users.md)
- **pubnub-security** — Functions sign with the [secret key from Vault](../pubnub-security/references/access-manager.md)
- **pubnub-reliability** — server-side [idempotency check via KVStore](../pubnub-reliability/references/idempotent-publish.md)
- **pubnub-observability** — [function execution metrics](../pubnub-observability/references/usage-metrics.md), [incident triage](../pubnub-observability/references/incident-runbook.md) for slow Functions
- **pubnub-choose-docs-path** — for routing other PubNub questions

## Output Format

When providing implementations:
1. Include complete, working function code **from this skill's patterns** when the user asks for deployable Functions code.
2. When the user asks from a **vertical context** (voting, auctions, betting, gaming), **link Pattern 1 here** and describe only the vertical delta — do not duplicate the canonical handler in the vertical skill's voice.
3. Show proper async/await with try/catch.
4. Explain module usage and imports.
5. Note channel pattern configuration.
6. Include deployment instructions (`manage_functions` when relevant).
7. State the function type at the top of every snippet.

Shared pattern routing for all skills: [shared-pattern-routing.md](../pubnub-choose-docs-path/references/shared-pattern-routing.md).
