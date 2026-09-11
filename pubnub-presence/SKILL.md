---
name: pubnub-presence
description: Real-time presence with PubNub. Covers Admin Portal Presence add-on configuration, join/leave/timeout events, hereNow occupancy, presence state, dropped-connection categories (PNNetworkDownCategory etc.), heartbeat tuning, and multi-device sync for the same userId. Use when implementing online/offline indicators, occupancy counts, last-seen tracking, or troubleshooting presence flapping.
license: PubNub
metadata:
  author: pubnub
  version: "0.3.0"
  domain: real-time
  triggers: pubnub, presence, online, offline, occupancy, status, users, hereNow, whereNow, withPresence, presence state, heartbeat, PNNetworkDownCategory, PNReconnectedCategory, multi-device
  role: specialist
  scope: implementation
  output-format: code
---

# PubNub Presence Specialist


> **Precedence:** PubNub MCP tools and pubnub.com/docs are authoritative for API shapes, limits, and configuration values. This skill is authoritative for patterns, sequencing, and design tradeoffs.


## Core Workflow

1. **Enable & scope** — Presence add-on + Presence Management channel rules ([presence-setup.md](references/presence-setup.md)).
2. **Subscribe with presence** — retrieve API from **`get_sdk_documentation`**.
3. **Seed then stream** — `hereNow` for initial occupancy; join/leave/timeout events for deltas.
4. **Tune heartbeat/timeout** — trade latency vs event volume ([presence-setup.md](references/presence-setup.md)).
5. **Handle disconnects** — [dropped-connections.md](references/dropped-connections.md) + [backoff](../pubnub-reliability/references/backoff-and-jitter.md).
6. **Multi-device** — [multi-device-sync.md](references/multi-device-sync.md).

## Reference Guide

| Reference | Purpose |
|-----------|---------|
| [presence-setup.md](references/presence-setup.md) | Portal prerequisites, heartbeat tuning |
| [presence-patterns.md](references/presence-patterns.md) | Scalable presence patterns |
| [dropped-connections.md](references/dropped-connections.md) | Status categories vs presence leave |
| [multi-device-sync.md](references/multi-device-sync.md) | userId design for multiple devices |

## Event orchestration

| Phase | Action |
|-------|--------|
| Initial | After connect, fetch occupancy (`hereNow`) before trusting empty UI |
| Live | Apply join/leave/timeout/state-change deltas — retrieve event shapes from SDK docs |
| High occupancy | Expect interval events; batch UI updates |
| Leave UX | Distinguish explicit leave vs timeout vs disconnect ([dropped-connections.md](references/dropped-connections.md)) |

## Constraints

- Channel must be allowed in Presence Management rules.
- Persistent [`userId`](../pubnub-app-developer/SKILL.md) required.
- Presence is **per-connection** — see [multi-device-sync.md](references/multi-device-sync.md).
- Mind event volume on large channels — [cost & payload hygiene](../pubnub-observability/references/cost-and-payload-hygiene.md).

## MCP Tools

- **`get_sdk_documentation`** — presence APIs, hereNow, state
- **`subscribe_and_receive_pubnub_messages`** — verify event flow in staging

## See Also

- **pubnub-app-developer** — pub/sub primitives
- **pubnub-app-context** — persistent user metadata alongside presence
- **pubnub-observability** — [presence flapping runbook](../pubnub-observability/references/incident-runbook.md)
- **pubnub-scale** — large-event occupancy
