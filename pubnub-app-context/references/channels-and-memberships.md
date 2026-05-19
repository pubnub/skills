<!-- canonical-for: APP_CONTEXT_CHANNELS_AND_MEMBERSHIPS -->
<!-- used-by: pubnub-chat -->

# App Context: Channels and Memberships

The canonical reference for storing channel metadata and tracking user-channel relationships in PubNub App Context.

## Channel Objects

A Channel Object stores metadata about a channel (a chat room, a topic, a logical group, an IoT device, etc.). The channel name is the identifier and matches the channel name used in pub/sub.

Standard fields:

| Field | Type | Purpose |
|---|---|---|
| `id` | string | Channel name. Required, unique. |
| `name` | string | Display name (can differ from id) |
| `description` | string | Description / topic |
| `custom` | object | Application-specific scalar fields |
| `eTag` | string | Server-managed concurrency token |
| `updated` | timestamp | Server-managed last-update time |

### Create / Update a Channel

```javascript
await pubnub.objects.setChannelMetadata({
  channel: 'room-photo-challenge-123',
  data: {
    name: 'Photo Challenge',
    description: 'Take a creative photo in 5 minutes',
    custom: {
      challenge_type: 'photo',
      created_by: 'user-456',
      duration_minutes: 5,
      created_at: '2026-04-29T16:32:00Z',
      max_participants: 20,
      schema_version: 1
    }
  }
});
```

### Read a Channel

Pass `include: { customFields: true }` to get `custom`.

```javascript
const result = await pubnub.objects.getChannelMetadata({
  channel: 'room-photo-challenge-123',
  include: { customFields: true }
});

console.log(result.data.name);                    // "Photo Challenge"
console.log(result.data.custom.challenge_type);   // "photo"
```

### List All Channels

```javascript
const result = await pubnub.objects.getAllChannelMetadata({
  include: { customFields: true, totalCount: true },
  filter: 'custom.challenge_type == "photo"',
  sort: { updated: 'desc' },
  limit: 50
});
```

### Delete a Channel

```javascript
await pubnub.objects.removeChannelMetadata({ channel: 'room-photo-challenge-123' });
```

If Referential Integrity is enabled, all memberships in this channel are also removed.

## Memberships

A Membership represents the relationship between a user and a channel. Each Membership can carry its own `custom` metadata (role, joined-at, permissions, status, mute-flag, etc.).

### Create a Membership (User Joins Channel)

```javascript
await pubnub.objects.setMemberships({
  uuid: 'user-123',
  channels: [
    {
      id: 'room-photo-challenge-123',
      custom: {
        joined_at: '2026-04-29T16:32:00Z',
        role: 'member',
        permissions: ['read', 'write'],
        status: 'active',
        muted: false
      }
    }
  ]
});
```

If `uuid` is omitted, the SDK uses the current PubNub [`userId`](../../pubnub-app-developer/references/sdk-patterns.md).

### Get a User's Memberships

```javascript
const result = await pubnub.objects.getMemberships({
  uuid: 'user-123',
  include: {
    customFields: true,
    channelFields: true,
    customChannelFields: true,
    totalCount: true
  },
  limit: 100
});

result.data.forEach(m => {
  console.log(`User is in: ${m.channel.name} as ${m.custom.role}`);
});
```

The `include` object lets you hydrate the embedded channel object inline so you don't need a second request per channel.

### Get a Channel's Members

```javascript
const result = await pubnub.objects.getChannelMembers({
  channel: 'room-photo-challenge-123',
  include: {
    customFields: true,
    UUIDFields: true,
    customUUIDFields: true,
    totalCount: true
  },
  limit: 100
});

result.data.forEach(m => {
  console.log(`Member: ${m.uuid.name} (role: ${m.custom.role})`);
});
```

### Remove a Membership (User Leaves Channel)

```javascript
await pubnub.objects.removeMemberships({
  uuid: 'user-123',
  channels: ['room-photo-challenge-123']
});
```

### Update a Membership

`setMemberships` is upsert — calling it again with the same (user, channel) pair overwrites the membership's custom data. To do a partial update, read first and merge:

```javascript
const current = await pubnub.objects.getMemberships({ uuid: 'user-123', include: { customFields: true } });
const existing = current.data.find(m => m.channel.id === 'room-photo-challenge-123');

await pubnub.objects.setMemberships({
  uuid: 'user-123',
  channels: [{
    id: 'room-photo-challenge-123',
    custom: { ...existing.custom, muted: true }
  }]
});
```

## Membership Patterns

### Roles

Encode roles in the membership `custom`, not in the channel `custom`. A user's role is per-channel (admin in one room, member in another).

```javascript
custom: { role: 'admin' | 'moderator' | 'member' | 'guest' }
```

### Notification Preferences

Per-user, per-channel mute settings live on the membership:

```javascript
custom: {
  muted: true,
  notification_level: 'mentions_only'   // or 'all', 'none'
}
```

### Audit / Provenance

```javascript
custom: {
  joined_at: '2026-04-29T16:32:00Z',
  invited_by: 'user-456'
}
```

### Soft Delete

If you want to keep history of past memberships, use a status flag instead of removing:

```javascript
custom: { status: 'inactive', left_at: '2026-04-29T18:45:00Z' }
```

## Listening for Channel and Membership Changes

For the underlying [`addListener` and `pubnub.subscribe` mechanics](../../pubnub-app-developer/references/publish-subscribe.md), see the canonical owner.

```javascript
pubnub.addListener({
  objects: (event) => {
    if (event.message.type === 'channel') {
      console.log('Channel event:', event.message.event, event.message.data.id);
    }
    if (event.message.type === 'membership') {
      console.log('Membership event:', event.message.event, event.message.data);
    }
  }
});

pubnub.subscribe({ channels: ['room-photo-challenge-123'] });
```

## Common Anti-Patterns

| Anti-pattern | Fix |
|---|---|
| Storing user roles on the channel object | Store on the membership (per-user, per-channel) |
| Calling `setMemberships` for partial update without reading first | Read + merge + set |
| Membership operations failing silently with Referential Integrity on | Ensure user object and channel object exist first |
| Listing all channels then filtering client-side | Use [filter expressions](metadata-and-filtering.md) server-side |
| Hard-deleting departing members | Use a `status: 'inactive'` soft delete if history matters |

## Related Reading

- [users.md](users.md) — User Objects (UUID Metadata)
- [metadata-and-filtering.md](metadata-and-filtering.md) — filter syntax, sorting, change events
