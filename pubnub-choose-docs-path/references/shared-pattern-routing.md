# Shared Pattern Routing (S1–S8)

When a user asks about one of these patterns from **any** skill, **route to the canonical owner** first. Non-owner skills keep **domain delta only** (tables, channel names, business rules) — not a second full implementation.

## Agent behavior (required)

1. **Link the owner file** for the pattern (below).
2. **State the vertical/domain delta** in prose or a short table (what differs for polls vs auctions vs sport).
3. **Do not paste** a full Before-Publish handler, catch-up loop, status `switch`, sharding implementation, or delta-sync class **unless the user explicitly asks for deployable code** or a `manage_functions` package.
4. For deploy requests, start from the **owner** implementation and apply the vertical delta on top.

## Pattern catalog

| ID | Pattern | Canonical owner | Owner file |
|----|---------|-----------------|------------|
| **S1** | Server-authoritative Before-Publish counter / validation | `pubnub-functions` | [functions-patterns.md](../../pubnub-functions/references/functions-patterns.md) (Pattern 1, Pattern 7 for rate limits) |
| **S2** | Offline catch-up + dedup-on-merge | `pubnub-history` | [offline-catch-up.md](../../pubnub-history/references/offline-catch-up.md) — dedup primitive: [dedup-on-merge.md](../../pubnub-reliability/references/dedup-on-merge.md) |
| **S3** | Reconnect + status-category wiring | `pubnub-reliability` | [backoff-and-jitter.md](../../pubnub-reliability/references/backoff-and-jitter.md), [dropped-connections.md](../../pubnub-presence/references/dropped-connections.md) |
| **S4** | Large-event checklist + fan-out / sharding | `pubnub-scale` | [large-events.md](../../pubnub-scale/references/large-events.md) |
| **S5** | Delta / sequence / snapshot state sync | `pubnub-multiplayer-gaming` | [gaming-state-sync.md](../../pubnub-multiplayer-gaming/references/gaming-state-sync.md) — reusable beyond gaming (sport feeds, IoT, dashboards) |
| **S6** | Subscribe-key rotation = keyset migration | `pubnub-keyset-management` | [key-rotation-and-hygiene.md](../../pubnub-keyset-management/references/key-rotation-and-hygiene.md) |
| **S7** | App Context eTag optimistic concurrency | `pubnub-app-context` | [users.md](../../pubnub-app-context/references/users.md#concurrency-use-etag) |
| **S8** | Multi-device `userId` tradeoff | `pubnub-presence` | [multi-device-sync.md](../../pubnub-presence/references/multi-device-sync.md) |

## Vertical delta examples (what stays local)

| Vertical | Pattern | Domain delta only (not a second canonical copy) |
|----------|---------|--------------------------------------------------|
| Live voting | S1 | Poll KV keys, `poll.*.votes` / `.results` channels, error codes, tally broadcast throttle |
| Live auctions | S1 | Bid validation rules, lock/CAS for concurrent bids, outbid notifications |
| Betting | S1 + downstream | Edge checks on `wagers.submit`; odds drift + balance in After Publish |
| Sport updates | S2, S5 | Catch-up on score/play channels; sequence on scoreboard; sport play-type filters |
| Gaming | S1, S5 | Move/speed rules, snapshot on reconnect, room channel layout |
| Chat / app-developer | S8 | Link to presence owner; chat-specific UX consequences only |

## Not shared patterns (do not consolidate here)

Per-language SDK init, push envelopes, generic secret hygiene, payload sizing tables — defer to MCP/docs (AIG-530) or observability/history owners.
