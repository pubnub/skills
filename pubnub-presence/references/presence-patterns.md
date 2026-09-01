<!-- canonical-for: PRESENCE_PATTERNS -->
<!-- used-by: -->

> **Cross-references:** [`new PubNub()` initialization, userId/UUID, `addListener` and pubnub.subscribe basics](../../pubnub-app-developer/references/sdk-patterns.md) (and [pub/sub patterns](../../pubnub-app-developer/references/publish-subscribe.md)). [Reconnect with backoff](../../pubnub-reliability/references/backoff-and-jitter.md), [dedup-on-merge](../../pubnub-reliability/references/dedup-on-merge.md), [multi-device sync](multi-device-sync.md), [App Context user metadata](../../pubnub-app-context/references/users.md).

# PubNub Presence Patterns

## Prerequisites

| Requirement | Why |
|-------------|-----|
| Persistent unique `userId` | Unstable IDs break presence and billing |
| Event Engine enabled | Better reconnect and status categories |
| Initial `hereNow` on connect | Events only report deltas, not baseline occupancy |
| Cleanup on unload/unmount | Without it, users appear online until timeout |

## hereNow optimization

Request only fields the UI needs — `includeUUIDs: false` and `includeState: false` when showing counts only. Cache occupancy briefly to avoid polling; rely on presence events after the initial snapshot.

## Scale decisions

| Use Case | Recommendation |
|----------|----------------|
| Chat room (< 100 users) | Full presence with user list |
| Chat room (100–1000 users) | Occupancy count only |
| Chat room (1000+ users) | Disable or sample; use interval events |
| Gaming lobby | Full presence for matchmaking |
| IoT device status | Full presence with custom state |
| Live event (10K+ users) | Disable individual presence; use aggregated counts |

## High-occupancy pattern

When occupancy exceeds **Announce Max** (configure in Admin Portal), presence switches to **interval** events with batched join/leave arrays instead of per-user events. Handle both `interval` and individual `join`/`leave`/`timeout`/`state-change` in the same listener.

## Multi-channel presence

Subscribe to presence only on channels that need it (`channelsWithPresence`). Batch `hereNow` across multiple channels in one call when bootstrapping dashboards.

## User state

Set state at subscribe time for avatar/status; update with `setState` when status changes. Read peer state via `getState` when building member lists — do not assume state arrives with every join event.

## Connection lifecycle

On `PNReconnectedCategory`, refresh presence baseline with `hereNow` — delta events during disconnect may have been missed. Pair status handling with [dropped-connections](dropped-connections.md) and [backoff-and-jitter](../../pubnub-reliability/references/backoff-and-jitter.md).

## Multi-device

Use the same authenticated `userId` on all devices; differentiate devices via custom state fields (`deviceType`, `deviceId`). See [multi-device-sync](multi-device-sync.md) for the tradeoff between shared vs per-device identity.

## Validation checklist

- [ ] Presence enabled in Admin Portal and channel rules configured
- [ ] Persistent `userId` and Event Engine
- [ ] Initial `hereNow` on connect; all event types handled
- [ ] Cleanup on page unload / component unmount
- [ ] Announce Max considered for high-occupancy channels
- [ ] hereNow fields minimized; no unnecessary polling
