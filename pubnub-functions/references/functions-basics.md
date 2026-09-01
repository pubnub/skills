<!-- canonical-for: FUNCTION_BASICS -->
<!-- used-by: -->

> **Cross-references:** Built on [pub/sub semantics](../../pubnub-app-developer/references/publish-subscribe.md) and [SDK initialization including userId/UUID](../../pubnub-app-developer/references/sdk-patterns.md). For [chained Functions](functions-chaining.md) and [DB-trigger patterns + runtime quirks](db-triggers-and-runtime-quirks.md). For non-transform routing prefer [Events & Actions](../../pubnub-events-and-actions/SKILL.md).

# PubNub Functions 2.0 Basics

## Function type selection

| Type | Use when |
|------|----------|
| **Before Publish** | Validate, enrich, or block messages before delivery |
| **After Publish** | Side effects that must not delay publish (analytics, webhooks) |
| **On Request** | HTTP endpoints backed by PubNub state |
| **On Interval** | Scheduled aggregation, cleanup, heartbeat jobs |

Each handler must return the trigger's completion call on every path: `request.ok()` / `request.abort()` for message events, `response.send(...)` for On Request, `event.ok()` / `event.abort()` for On Interval.

## Channel patterns (Before/After Publish)

Retrieve current wildcard rules via **`manage_functions`** / Functions documentation before binding handlers.

| Rule | Decision |
|------|----------|
| Wildcard position | `*` must be at the **end** of the pattern |
| Wildcard depth | Wildcard Subscribe patterns: max **two dots** in the pattern (`a.*`, `a.b.*`) — does **not** cap plain channel names |
| Delimiter | Period (`.`) is the hierarchy delimiter for bindings |

## On Request URI routing

On Request uses **URI path matching**, not channel wildcards.

| Rule | Implication |
|------|-------------|
| Match style | Character-by-character path comparison |
| `*` wildcard | Matches 0..N characters; multiple `*` allowed |
| Slashes | Not special; `/users` ≠ `/users/` |
| Params | Wildcard matches do **not** populate `request.params` — parse `request.path` |
| Portal UI | Enter pattern **without** leading `/`; requests arrive **with** `/` |
| Overlap | Overlapping patterns rejected at config time |

**Single-router pattern (recommended):** One Function with prefix pattern (e.g. `v1/tasks*`) and dispatch by `request.method` + parsed `request.path` segments inside the handler. Avoids overlap rejection when many endpoints share a prefix.

## Async orchestration

| Rule | Why |
|------|-----|
| Prefer `async/await` + `try/catch` | Clear error paths and completion returns |
| Return promise chains if used | Handler must return the chain or runtime exits early |
| Parallel independent I/O | `Promise.all` for vault + kvstore + xhr when budgets allow |
| Secrets | Always `vault.get`; never hardcode |

## Execution budgets

Per-module limits (XHR, KV Store, PubNub API, Vault) are **independent** — not a shared pool. Chain depth and timeout are platform limits. Retrieve current defaults and raise limits via Support using **`manage_functions`** / Functions docs.

When approaching limits: prioritize kvstore for hot paths, use `fire` for analytics that must not hit subscribers, and split heavy fan-out across [chained Functions](functions-chaining.md) within depth cap.

## Logging and validation

Logs appear in Admin Portal under Functions → Your Function → Logs. Test in staging with representative payloads before enabling production channel patterns.

## Deployment sequence

`understand → configure patterns → implement → validate budgets/secrets → staging test → production enable → monitor logs`
