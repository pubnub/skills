<!-- canonical-for: MULTI_DEVICE_SYNC -->
<!-- used-by: pubnub-reliability, pubnub-app-context -->

# Multi-Device Sync: Same User on Multiple Devices

The canonical reference for handling presence and message delivery when one user has multiple devices (phone + laptop + tablet) connected simultaneously.

> **Cross-references:** [`userId` semantics](../../pubnub-app-developer/references/sdk-patterns.md) is the primary lever. [Dedup-on-merge](../../pubnub-reliability/references/dedup-on-merge.md) handles cross-device duplicate suppression. [App Context user metadata](../../pubnub-app-context/references/users.md) tracks per-user state across devices.

## Two Choices: Shared `userId` vs Per-Device `userId`

| Choice | Implication |
|---|---|
| **Shared `userId`** (e.g., `user-123` across all devices) | Single presence entry per channel; same identity across devices |
| **Per-device `userId`** (e.g., `user-123-iphone`, `user-123-laptop`) | Each device is a distinct presence entry; separate identities |

## When to Use Shared `userId`

- The application semantics are "the user is here" — chat occupancy, "online" indicator
- One subscription per user across devices (saves transactions on a per-user pricing model)
- Simpler [App Context user record](../../pubnub-app-context/references/users.md)

### Behavior with Shared `userId`

- Subscribing the same `userId` to the same channel from two devices results in **one presence entry** in `hereNow`.
- The first `join` event fires; the second device's subscribe is silent for presence.
- `leave` fires only when the **last** device disconnects.
- Both devices receive every message — fan-out doubles per multi-device user.

### Caveats

- "User typing" indicator: if both devices show typing, the receiver sees two typing notifications (or one, depending on dedup).
- Last-seen tracking: leaves only when ALL devices leave, which may be hours later.

## When to Use Per-Device `userId`

- The application needs per-device state (which device is active, where mouse is, last action by device)
- Per-device push notification suppression (don't push to a device that's actively connected)
- Distinct online indicators per device

### Behavior with Per-Device `userId`

- Each device is a separate presence entry. `hereNow` shows N entries per real user.
- Each device fires its own `join` and `leave`.
- Application UI must aggregate "any of this user's devices online" → user online.

### Caveats

- Higher presence event volume (every device transition fires events).
- More transactions for fan-out (each device gets its own copy).
- App Context user metadata still keyed by the real user, not the per-device id.

## userId Construction Patterns

### Shared

```javascript
function getUserId() {
  return currentUser.id;     // e.g., "user-123"
}
```

### Per-Device with Real-User Suffix

```javascript
function getUserId() {
  const realUser = currentUser.id;
  let deviceId = localStorage.getItem('device_id');
  if (!deviceId) {
    deviceId = crypto.randomUUID().slice(0, 8);
    localStorage.setItem('device_id', deviceId);
  }
  return `${realUser}.${deviceId}`;
}

function realUserOf(pubnubUserId) {
  return pubnubUserId.split('.')[0];
}
```

The `.` separator lets you cheaply extract the real user from any PubNub event.

## Cross-Device Message Dedup

When the same logical message gets published to a channel that two of your devices are subscribed to, both devices receive it. The application UI may want to:

- Show it once, regardless of which device received it first
- Sync the "read" state across devices

For the first, dedup by `message_id` (client-generated UUID). See [pubnub-reliability/references/dedup-on-merge.md](../../pubnub-reliability/references/dedup-on-merge.md).

For the second, persist read state in [App Context membership metadata](../../pubnub-app-context/references/channels-and-memberships.md) and sync via [metadata change events](../../pubnub-app-context/references/metadata-and-filtering.md).

## Push vs In-App Delivery

A common pattern: send push notification to mobile devices that are NOT actively connected; rely on real-time delivery to active devices.

| Tactic | How |
|---|---|
| Detect active devices | Use [presence](presence-events.md) `hereNow` or active-device tracking in App Context |
| Suppress push if active in-app | Server checks active devices before triggering push |
| Per-device opt-out | Per-device `userId` lets you target push by device |

This is hard to get fully right; expect occasional duplicate notifications.

## "User Online" Indicator with Per-Device `userId`

```javascript
function isUserOnline(realUserId, hereNowResult) {
  return hereNowResult.uuids
    .some(uuid => realUserOf(uuid) === realUserId);
}
```

Aggregate the per-device entries into a per-user view at the application layer.

## Anti-Patterns

| Anti-pattern | Fix |
|---|---|
| Random `userId` per session | Use a persistent userId; see [sdk-patterns.md](../../pubnub-app-developer/references/sdk-patterns.md) |
| Shared `userId` everywhere when device-specific state is needed | Switch to per-device with real-user suffix |
| Per-device `userId` when "is this user online" is the only question | Use shared; saves cost and complexity |
| Forgetting cross-device message dedup | Always dedup by `message_id` |
| Sending push to all devices when one is in-app | Suppress push for active devices |
| Mixing schemes (shared on web, per-device on mobile) | Pick one and apply consistently |

## Related Reading

- [presence-events.md](presence-events.md) — `join`, `leave`, `hereNow` semantics
- [dropped-connections.md](dropped-connections.md) — connection state per device
- [presence-patterns.md](presence-patterns.md) — additional patterns
- [pubnub-app-developer/references/sdk-patterns.md](../../pubnub-app-developer/references/sdk-patterns.md) — `userId` rules
- [pubnub-reliability/references/dedup-on-merge.md](../../pubnub-reliability/references/dedup-on-merge.md) — message dedup
