<!-- canonical-for: EVENTS_AND_ACTIONS -->
<!-- used-by: pubnub-choose-docs-path, pubnub-illuminate, pubnub-presence, pubnub-app-context -->

# Events & Actions: Event Catalog

The canonical reference for all event sources, event types, and payload schemas exposed by PubNub Events & Actions.

## Five Event Sources

A listener picks exactly one event source. Then it picks one event type within that source.

| Event source | Triggers on |
|---|---|
| Messages | Pub/Sub publishes, message reactions, file uploads |
| Users | User CRUD (App Context user metadata) and presence state changes |
| Channels | Subscribe / unsubscribe / occupancy events and channel CRUD |
| Mobile Push notifications | Push delivery errors and device removals |
| Memberships | Membership CRUD (App Context membership metadata) |

## Messages Source

| Producer | Event type | Description |
|---|---|---|
| Pub/Sub | Message sent | A message was published on a channel (excludes internal `-pnpres` channels) |
| Message Reaction | Message reaction created | A reaction was added |
| Message Reaction | Message reaction deleted | A reaction was removed |
| Files | File sent | A file was sent to a channel |

For Message Reactions semantics see [pubnub-chat/references/message-actions.md](../../pubnub-chat/references/message-actions.md). For Files see [pubnub-chat/references/file-sharing.md](../../pubnub-chat/references/file-sharing.md).

### Sample Message Publish Payload

```json
{
  "meta": { /* metadata param if any */ },
  "message": { /* publish payload */ },
  "channel": "chat-room",
  "uuid": "user-123"
}
```

## Users Source

| Producer | Event type | Description |
|---|---|---|
| Presence | User state changed in channel | Presence state changed on a channel |
| CRUD | User created | User metadata was set |
| CRUD | User updated | User metadata was changed |
| CRUD | User deleted | User metadata was removed |

CRUD events here correspond to [App Context user metadata](../../pubnub-app-context/references/users.md) operations.

## Channels Source

| Producer | Event type | Description |
|---|---|---|
| Presence | User started subscription to channel | A user subscribed |
| Presence | User stopped subscription to channel | A user unsubscribed |
| Presence | User timed out while subscribing to channel | Subscription timed out |
| Presence | First user subscribed to channel | Occupancy 0→1 |
| Presence | Last user left channel | Occupancy 1→0 |
| Presence | Interval occupancy counted | Periodic occupancy snapshot |
| CRUD | Channel created | Channel metadata set |
| CRUD | Channel updated | Channel metadata changed |
| CRUD | Channel deleted | Channel metadata removed |

For [Presence event semantics](../../pubnub-presence/references/presence-events.md) and CRUD see [App Context channel metadata](../../pubnub-app-context/references/channels-and-memberships.md).

## Mobile Push Notifications Source

| Producer | Event type | Description |
|---|---|---|
| Devices | Device removed | Device token removed from a channel |
| Devices | Push error | Push notification delivery error |

## Memberships Source

| Producer | Event type | Description |
|---|---|---|
| CRUD | Membership created | Membership metadata set |
| CRUD | Membership updated | Membership metadata changed |
| CRUD | Membership deleted | Membership metadata removed |

CRUD events here correspond to [App Context membership metadata](../../pubnub-app-context/references/channels-and-memberships.md) operations.

## Webhook Payload Schemas

Every webhook payload includes a top-level `schema` field that uniquely identifies the event type and version:

```
pubnub.com/schemas/events/<event-name>?v=<version>
```

Receivers should switch on `schema` to handle each event type:

```javascript
function handleE&AWebhook(req, res) {
  const events = req.body.data || [];
  for (const event of events) {
    const schema = req.body.schema;
    switch (true) {
      case schema.includes('presence.user.channel.joined'):     onJoin(event); break;
      case schema.includes('presence.user.channel.left'):       onLeave(event); break;
      case schema.includes('presence.user.timedout'):           onTimeout(event); break;
      case schema.includes('presence.channel.state.active'):    onChannelActive(event); break;
      case schema.includes('presence.channel.state.inactive'):  onChannelInactive(event); break;
      case schema.includes('push.device.removed'):              onDeviceRemoved(event); break;
      case schema.includes('push.message.sending.failed'):      onPushError(event); break;
      default:                                                  onUnknown(event);
    }
  }
  res.sendStatus(200);
}
```

Sample payloads for the most common events:

### Presence: User Joined

```json
{
  "schema": "pubnub.com/schemas/events/presence.user.channel.joined?v=1.0.0",
  "data": [
    {
      "id": "1d558c0f-78d8-576a47bb0658",
      "channel": "CHANNEL-NAME",
      "userId": "5934fbc6-bf06b3c3b365",
      "occupancy": 3,
      "data": {},
      "timestamp": "2026-04-29T10:00:20.021Z",
      "subKey": "SUBKEY-HERE"
    }
  ]
}
```

### Presence: User Left

```json
{
  "schema": "pubnub.com/schemas/events/presence.user.channel.left?v=1.0.0",
  "data": [{ "id": "...", "channel": "...", "userId": "...", "occupancy": 2, "timestamp": "..." }]
}
```

### Presence: Timeout

```json
{
  "schema": "pubnub.com/schemas/events/presence.user.timedout.in.channel?v=1.0.0",
  "data": [{ "id": "...", "channel": "...", "userId": "...", "occupancy": 1, "timestamp": "..." }]
}
```

### Presence: Channel Active (0→1)

```json
{
  "schema": "pubnub.com/schemas/events/presence.channel.state.active?v=1.0.0",
  "data": [{ "id": "...", "subKey": "...", "channel": "..." }]
}
```

### Presence: Interval Occupancy

```json
{
  "schema": "pubnub.com/schemas/events/presence.channel.occupancy.counted?v=1.0.0",
  "data": [
    {
      "id": "...",
      "channel": "CHANNEL-NAME",
      "occupancy": "3",
      "timestamp": "...",
      "subKey": "...",
      "usersJoined": ["id-1", "id-2"],
      "usersLeft":   ["id-3"]
    }
  ]
}
```

### Mobile Push: Error

```json
{
  "schema": "pubnub.com/schemas/events/push.message.sending.failed?v=1.0.0",
  "data": [
    {
      "id": "...",
      "channel": "...",
      "devices": "...",
      "platform": "APNS",
      "timestamp": "...",
      "state": "error",
      "subKey": "...",
      "payload": "ERROR MESSAGE"
    }
  ]
}
```

### Mobile Push: Device Removed

```json
{
  "schema": "pubnub.com/schemas/events/push.device.removed?v=1.0.0",
  "data": [
    {
      "id": "...",
      "action": "feedback|remove|update",
      "device": "...",
      "platform": "APNS",
      "timestamp": "...",
      "subKey": "..."
    }
  ]
}
```

For legacy webhook payloads and migration to v2.1, see the PubNub migration guide.

## userId / UUID Note

User ID is referred to as both `userId` and `uuid` in different parts of the API and documentation. They hold the same value — the [`userId` set during PubNub initialization](../../pubnub-app-developer/references/sdk-patterns.md).

## Related Reading

- [action-targets.md](action-targets.md) — where to send these events
- [filters-and-jsonpath.md](filters-and-jsonpath.md) — narrowing which events trigger
- [retries-and-batching.md](retries-and-batching.md) — delivery configuration
