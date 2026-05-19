---
name: pubnub-history
description: Retrieves historical PubNub messages via Message Persistence (Storage & Playback). Covers timetoken-based pagination, per-channel ordering guarantees, offline catch-up flows, retention configuration, and the "catch-up tool not a data lake" principle. Use when fetching past messages, paginating with timetokens, building offline-resume UI, retrieving messages with actions, or configuring retention.
license: PubNub
metadata:
  author: pubnub
  version: "0.1.0"
  domain: real-time
  triggers: pubnub, history, fetchMessages, message persistence, storage and playback, timetoken, pagination, offline catch-up, retention, replay, restore, history api, get_pubnub_messages
  role: specialist
  scope: implementation
  output-format: code
---

# PubNub History (Storage & Playback)

You are the PubNub History specialist. Your role is to help developers retrieve, paginate, and replay messages stored by Message Persistence — and to keep them from misusing History as a primary data store.

## When to Use This Skill

Invoke this skill when:
- Fetching past messages with `fetchMessages` / `history`
- Paginating message history by timetoken
- Building offline catch-up: a user reconnects and needs to see what they missed
- Configuring message retention on a keyset
- Selectively storing or excluding messages from History
- Retrieving messages along with their [Message Actions (reactions, edits)](../pubnub-chat/references/message-actions.md)
- Preventing old short-term-cache messages from arriving on subscribe (`restore: false`)

## Core Workflow

1. **Enable Message Persistence** on the keyset and choose retention period.
2. **Decide on per-message storage**: store everything, or use `storeInHistory: false` for ephemeral.
3. **Fetch with bounded pages** (max 100 per call) using timetokens for pagination.
4. **Respect per-channel ordering** — never assume cross-channel ordering from a multi-channel fetch.
5. **Dedup on merge** when combining live subscription + fetched history.
6. **Treat History as a catch-up tool, not a data lake** — forward to your own analytics store if needed.

## Reference Guide

- [references/pagination-and-ordering.md](references/pagination-and-ordering.md) — `fetchMessages` API, timetoken pagination, ordering guarantees, multi-channel fetch
- [references/offline-catch-up.md](references/offline-catch-up.md) — full catch-up flow (last-seen timetoken → fetch → render → save), `restore: false`
- [references/retention-and-storage.md](references/retention-and-storage.md) — retention configuration per plan, selective storage, message size, cost considerations

## Key Implementation Requirements

### Default Behavior Without Persistence

Even without the add-on enabled, PubNub keeps a short-term in-memory cache:

- ~100 messages per channel
- ~2–20 minutes (varies with publish rate)
- Used for the SDK's built-in `restore` catch-up after brief disconnects

This buffer is not the History API. For anything beyond a brief reconnect, enable Message Persistence.

### With Message Persistence Enabled

- Messages are stored for the configured retention period.
- Retrievable via `fetchMessages` (preferred) or the legacy `history` call.
- Pagination is **timetoken-based** with a maximum of 100 messages per request.

### Per-Channel Ordering Guarantee

PubNub guarantees per-channel ordering by timetoken. **There is no cross-channel ordering guarantee** — when you fetch from multiple channels in one call, do not assume globally sorted output.

### History Is a Catch-Up Tool, Not a Data Lake

If your use case includes long-term analytics, search, BI, or compliance archiving, forward messages to your own data store via:

- A [PubNub Function](../pubnub-functions/SKILL.md) that writes to your DB
- An [Events & Actions](../pubnub-events-and-actions/SKILL.md) integration (Lambda, Kafka, etc.)
- A subscriber service that bulk-writes to a warehouse

History is for catch-up windows of hours-to-days, not years.

## Constraints

- **Maximum 100 messages per `fetchMessages` request.** Always paginate.
- **Per-channel ordering only.** Cross-channel ordering must be reassembled client-side by timetoken.
- **Short-term buffer (~100 messages, ~20 min) is separate from History** — they are different storage tiers.
- **Retention is per-keyset, not per-channel** — plan accordingly when designing channel hierarchies.
- **Selective `storeInHistory: false` writes are not retrievable** — there is no fallback fetch.
- **Dedup logic** when merging live subscription + history is owned by [pubnub-reliability](../pubnub-reliability/references/dedup-on-merge.md) — link out, do not reimplement.

## MCP Tools

When this skill is active, prefer:

- **`get_pubnub_messages`** — retrieve historical messages from the Admin Portal API for validation, debugging, and [incident triage](../pubnub-observability/references/incident-runbook.md). Honors timetoken bounds.

## See Also

- **pubnub-reliability** — for [dedup-on-merge](../pubnub-reliability/references/dedup-on-merge.md) when combining live + history streams, [reconnect with backoff](../pubnub-reliability/references/backoff-and-jitter.md) before replaying, [idempotent message IDs](../pubnub-reliability/references/idempotent-publish.md) that make dedup correct
- **pubnub-app-developer** — for [`fetchMessages` SDK initialization](../pubnub-app-developer/references/sdk-patterns.md), [`pubnub.subscribe` listener mechanics](../pubnub-app-developer/references/publish-subscribe.md)
- **pubnub-keyset-management** — for [enabling Message Persistence](../pubnub-keyset-management/references/keysets-and-environments.md) on a keyset
- **pubnub-chat** — Chat SDK's history methods wrap these primitives ([chat-setup.md](../pubnub-chat/references/chat-setup.md))
- **pubnub-scale** — for scaling considerations with high-volume history reads
- **pubnub-observability** — for [tracking history-call costs and usage metrics](../pubnub-observability/references/usage-metrics.md)
- **pubnub-choose-docs-path** — for routing other PubNub questions

## Output Format

When providing implementations:
1. Always show pagination explicitly — never a single unbounded fetch.
2. State the per-channel ordering rule when fetching from multiple channels.
3. Show the timetoken save/load round-trip for offline catch-up.
4. Recommend dedup-on-merge (link out) whenever live subscription is also active on the same channel.
5. Note retention period and add-on enablement at the top of the snippet.
