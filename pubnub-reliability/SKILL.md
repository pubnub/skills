---
name: pubnub-reliability
description: Cross-cutting reliability patterns for PubNub apps. Covers reconnect with exponential backoff + jitter, idempotent publish with client-generated message IDs, dedup-on-merge for live + history streams, queue-and-retry for offline writes, and schema versioning of message envelopes. Use during design reviews, when planning offline support, or during incident response when network or delivery reliability is the concern.
license: PubNub
metadata:
  author: pubnub
  version: "0.1.0"
  domain: real-time
  triggers: pubnub, reliability, retry, backoff, jitter, idempotent, dedup, deduplication, queue, schema version, message envelope, exponential backoff, offline, reconnect strategy
  role: specialist
  scope: implementation
  output-format: code
---

# PubNub Reliability Patterns

You are the PubNub reliability specialist. Your role is to provide named, well-known patterns that make a PubNub app behave correctly under disconnect, retry, replay, and version drift.

## When to Use This Skill

Invoke this skill when:
- Planning offline support for a mobile or web client
- Designing reconnect behavior for an SDK that exposes its own retry knobs
- Eliminating duplicate-message bugs after reconnect
- Combining live subscription with historical fetch (the most common dedup scenario)
- Versioning the JSON shape of messages across client releases
- Investigating an incident where messages were delivered twice, lost, or out-of-order

This skill is **cross-cutting**. It applies to chat, IoT, gaming, finance, anything. Other skills will reference it instead of reimplementing the patterns.

## Core Workflow

For every persistent connection you operate, decide:

1. **Reconnect strategy**: backoff + jitter, with bounded max retries before giving up. See [references/backoff-and-jitter.md](references/backoff-and-jitter.md).
2. **Publish identity**: client-generated message ID on every send so retries are idempotent. See [references/idempotent-publish.md](references/idempotent-publish.md).
3. **Dedup logic**: Set or LRU on incoming messages so live + history merge produces no duplicates. See [references/dedup-on-merge.md](references/dedup-on-merge.md).
4. **Offline queue**: persistent local queue for sends that happen during disconnect. See [references/queue-and-retry.md](references/queue-and-retry.md).
5. **Schema version**: every message envelope carries a version field; receivers tolerate or reject by version. See [references/schema-versioning.md](references/schema-versioning.md).

## Reference Guide

- [references/backoff-and-jitter.md](references/backoff-and-jitter.md) — exponential backoff with full jitter, max-attempts cap, listening for connection state
- [references/idempotent-publish.md](references/idempotent-publish.md) — client-generated `message_id`, server-side dedup with PubNub Functions
- [references/dedup-on-merge.md](references/dedup-on-merge.md) — Set-based dedup, LRU, dedup-by-timetoken vs dedup-by-message-id, when to use each
- [references/queue-and-retry.md](references/queue-and-retry.md) — persistent client queue, retry policy, drain ordering, observability
- [references/schema-versioning.md](references/schema-versioning.md) — envelope shape, version field, forward and backward compatibility, deprecation flow

## Key Implementation Requirements

### The Five Reliability Patterns

Every robust PubNub app has all five. Skipping any one creates a class of bugs that's hard to diagnose later.

| Pattern | Bug class it prevents |
|---|---|
| Backoff + jitter | Thundering-herd reconnect storms after a regional outage |
| Idempotent publish | Duplicate messages from network retries |
| Dedup on merge | Duplicate messages from live + history overlap |
| Queue and retry | Lost messages published while offline |
| Schema versioning | Old clients crash on new fields, new clients can't read old data |

### When to Apply Which

| Scenario | Patterns required |
|---|---|
| Read-only subscriber (live ticker) | Backoff + jitter |
| Chat client (publish + subscribe) | All five |
| IoT publisher with intermittent connectivity | Backoff + jitter, idempotent publish, queue + retry, schema versioning |
| Mobile app with offline support | All five |
| Server-to-server pipeline | Idempotent publish, dedup on merge, schema versioning |
| Real-time game | Backoff + jitter, schema versioning, dedup if joining mid-session |

## Constraints

- **Backoff + jitter must always include random jitter.** A pure exponential backoff still synchronizes if every client computed the same delay from the same outage start time.
- **Idempotent publish requires the message ID be deterministic at the source** (not regenerated on retry).
- **Dedup state must outlive the connection.** A naive dedup `Set` on the live listener doesn't handle cold-start replay; persist or rebuild it from history.
- **Offline queue must be persistent** (localStorage / SQLite / IndexedDB), not in-memory.
- **Schema version is immutable per message.** Never reuse a version number for a different shape.
- **Reconnect strategy is per-connection.** Don't share retry state across PubNub instances.

## MCP Tools

This skill is design-and-pattern oriented. No MCP tool is required for the patterns themselves. Apply patterns through the SDK and verify with:

- **`get_pubnub_messages`** — for [history-based dedup verification](../pubnub-history/references/pagination-and-ordering.md)
- **`subscribe_and_receive_pubnub_messages`** — for live test
- **`send_pubnub_message`** — for retry/idempotency test

## See Also

- **pubnub-app-developer** — for the underlying [`new PubNub` initialization](../pubnub-app-developer/references/sdk-patterns.md), [`pubnub.publish` and `pubnub.subscribe` mechanics](../pubnub-app-developer/references/publish-subscribe.md)
- **pubnub-history** — [dedup-on-merge](../pubnub-reliability/references/dedup-on-merge.md) goes hand in hand with [history fetch + live merge](../pubnub-history/references/offline-catch-up.md)
- **pubnub-presence** — [`PNNetworkDownCategory` / `PNReconnectedCategory`](../pubnub-presence/references/dropped-connections.md) drives backoff state
- **pubnub-functions** — server-side idempotency check via a [Function](../pubnub-functions/references/functions-basics.md) that consults [KV Store](../pubnub-functions/references/functions-modules.md)
- **pubnub-observability** — [logging correlation fields](../pubnub-observability/references/logging-correlation.md) include the message ID used by idempotent publish; [incident runbook](../pubnub-observability/references/incident-runbook.md) calls out reliability checks
- **pubnub-choose-docs-path** — for routing other PubNub questions

## Output Format

When providing implementations:
1. Recommend the reliability patterns relevant to the scenario; don't just answer the literal question.
2. Show realistic backoff numbers (200ms initial, 30s cap, full jitter).
3. Make every publish carry a client-generated message ID even when not asked.
4. Always recommend persistent storage for any offline queue.
5. Include schema-version field in every example envelope.
