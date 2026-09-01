# PubNub Functions 2.0 Modules

## Module selection

| Module | Use when |
|--------|----------|
| `kvstore` | Cross-execution state, counters, rate limits, chain handoff |
| `xhr` | External APIs, webhooks, enrichment |
| `vault` | Secrets (API keys, webhook URLs, JWT signing keys) |
| `pubnub` | Publish, signal, fire from inside the pipeline |
| `crypto` | HMAC, hashing (not a substitute for message CryptoModule) |
| `jwt` | Create/verify tokens at the edge |
| `uuid` | Generate correlation IDs |
| `jsonpath` | Extract fields from variable nested payloads |
| `ugc` | Lightweight built-in moderation (no XHR budget) |
| `codec/*` | Base64, query string, auth helpers |
| `advanced_math` | Geospatial / math helpers consistent with PubNub runtime |

## publish vs fire vs signal

| Method | Subscribers | Triggers Functions | Stored in History |
|--------|-------------|-------------------|-------------------|
| `publish` | Yes | Yes | Yes (if persistence on) |
| `fire` | No | Yes | No |
| `signal` | Yes | Yes | No |

Use **`fire`** for analytics or internal pipeline events that must not fan out to clients. Use **`signal`** for lightweight ephemeral client updates (typing, cursors).

## KVStore patterns

| Pattern | Rule |
|---------|------|
| TTL state | Set TTL on ephemeral keys (sessions, rate windows) |
| Counters | Use `incrCounter` / `getCounter` — not `set`/`get` |
| Rate limiting | Key by user + time window; increment atomically; abort when over limit |
| Chain handoff | Pass state between chained Functions via kvstore keys |

## Vault

Never hardcode secrets. Vault reads have an independent per-execution budget — retrieve current cap via **`how_to`** or Functions docs. Configure secrets in Admin Portal → Functions → Module → My Secrets.

## XHR vs UGC vs external moderation

| Approach | When |
|----------|------|
| `ugc` module | Lightweight checks without external API call (saves XHR budget) |
| `xhr` + external API | Full toxicity scoring, multi-category classification ([functions-patterns](functions-patterns.md)) |
| Abort vs flag | Before Publish: usually `request.abort()` on block; After Publish webhooks often `request.ok()` even if webhook fails |

## Budget-aware assembly

Plan module usage per handler: count expected `xhr.fetch`, `pubnub.*`, `kvstore.*`, and `vault.get` calls before deploy. Helpers like `crypto`, `uuid`, `jsonpath`, and most `codec/*` operations do not consume another module's budget — see [runtime quirks](db-triggers-and-runtime-quirks.md).

## Module surface

Exact method signatures and runtime-specific helpers change by Functions version. Use **`manage_functions`** and PubNub Functions documentation for the authoritative module API — do not rely on copied examples in Skills.
