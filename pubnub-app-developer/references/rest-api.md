<!-- canonical-for: REST_API -->
<!-- used-by: -->

> **Cross-references:** For [secret key, subscribe key, publish key](../../pubnub-keyset-management/references/keysets-and-environments.md) handling see the canonical owner.

# PubNub REST API: When to Use vs SDK

The canonical reference for the raw PubNub REST API — what it is, when you actually want it, and what to know if you do.

## Default: Use the SDK

For 95% of PubNub use cases, **use the official SDK** for your language. The SDKs handle:

- Connection multiplexing
- Reconnect with backoff
- Heartbeat and presence
- Request signing for Access Manager
- Encryption / decryption
- Rate limiting and retry
- Subscribe-streaming protocol (HTTP long-polling, WebSocket)

Reimplementing these in raw HTTP is slow, error-prone, and unnecessary.

## When the REST API Is Appropriate

| Case | Reason |
|---|---|
| Server-side language with no PubNub SDK | Make HTTP calls until an SDK exists |
| Embedded device too small for the SDK runtime | Direct HTTP to publish telemetry |
| One-off `curl` for debugging | Quick verify without writing code |
| Backend "fire-and-forget" publish from a serverless function with cold-start sensitivity | Skip SDK init time |
| Interop with a non-supported environment | Same as the no-SDK case |

For most server-side use, **the SDK is still better** even if the SDK has overhead — modern serverless cold starts handle it.

## Endpoint Base

```
https://ps.pndsn.com/
```

Or your [custom origin](../../pubnub-keyset-management/references/custom-origin.md) if configured.

## Authentication

- Most endpoints accept the publish or subscribe key in the URL path.
- Operations protected by [Access Manager](../../pubnub-security/references/access-manager.md) require an `auth` query parameter with a token.
- Admin operations require signature verification with the secret key (server-side only).

## Common REST Operations

### Publish

```
POST /publish/{pub_key}/{sub_key}/0/{channel}/0
Content-Type: application/json
{ "your": "message" }
```

Or as GET:

```
GET /publish/{pub_key}/{sub_key}/0/{channel}/0/<URL-encoded-JSON>
```

### Subscribe (long-poll)

```
GET /v2/subscribe/{sub_key}/{channels}/0?tt={timetoken}&uuid={userId}
```

Long-polls for up to ~280 seconds. Reconnect with the returned `t.t` as the next `tt`.

### History (`fetchMessages`)

```
GET /v3/history/sub-key/{sub_key}/channel/{channels}?max=100&start={timetoken}
```

For full history semantics see [pubnub-history/references/pagination-and-ordering.md](../../pubnub-history/references/pagination-and-ordering.md).

### Time

```
GET /time/0
```

Returns the current PubNub server timetoken. Useful for verifying clock alignment.

## Subscribe-Streaming Protocol Caveat

The SDK's subscribe loop is more nuanced than a single HTTP call:

- It maintains a long-lived HTTP request that returns when there are messages.
- On return, it reconnects immediately with the new timetoken.
- It handles network interruptions, reconnects, and heartbeats.
- It coordinates with the [presence](../../pubnub-presence/references/presence-events.md) endpoint.

If you DIY this in raw HTTP, expect to reimplement most of the [reliability patterns](../../pubnub-reliability/SKILL.md): backoff, dedup, queue, etc. **Strongly prefer the SDK for subscribers.**

## Authenticating with Access Manager

For operations covered by Access Manager, append `auth=<token>` to the query string:

```
POST /publish/{pub_key}/{sub_key}/0/restricted-channel/0?auth=<token>
```

Tokens are minted server-side via [`grantToken`](../../pubnub-security/references/access-manager.md). Never use the `secretKey` from a client.

## Rate Limiting

Same per-keyset rate limits apply via REST as via SDK. Heavy direct-HTTP use can trip [DoS protection](../../pubnub-security/references/dos-mitigation.md) — coordinate with PubNub Support if you expect very high volume.

## Versioning

PubNub REST endpoints have version prefixes (`/v2`, `/v3`). Version mixing is allowed but stick to a consistent set per integration. Newer versions deprecate older endpoints; check the docs for current recommended versions.

## Common Anti-Patterns

| Anti-pattern | Fix |
|---|---|
| DIY subscribe loop in raw HTTP for a supported language | Use the SDK |
| Hardcoding timetokens that drift | Always use the SDK's connection state or fetch via `/time/0` |
| Putting `secretKey` in client REST calls | Server-side only; mint tokens instead |
| Skipping URL encoding on JSON in GET path | Always URL-encode |
| Polling subscribe instead of long-polling | Honor the long-poll behavior |
| Not handling 5xx with backoff | Apply [backoff + jitter](../../pubnub-reliability/references/backoff-and-jitter.md) |

## Related Reading

- [sdk-patterns.md](sdk-patterns.md) — the recommended path
- [publish-subscribe.md](publish-subscribe.md) — the higher-level abstraction
- [pubnub-reliability/SKILL.md](../../pubnub-reliability/SKILL.md) — what you'll have to reimplement DIY
