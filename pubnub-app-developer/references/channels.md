<!-- canonical-for: CHANNEL_NAMING -->
<!-- used-by: pubnub-app-context, pubnub-scale, pubnub-chat -->

# PubNub Channels Guide

> **Cross-references:** Channel access control via [Access Manager](../../pubnub-security/references/access-manager.md). Channel storage via [Message Persistence](../../pubnub-history/references/pagination-and-ordering.md). For [Channel Groups, Stream Controller, channelGroups.addChannels](../../pubnub-scale/references/scaling-patterns.md) and [Wildcard Subscribe / Wildcard pattern](../../pubnub-scale/references/scaling-patterns.md) see the canonical owner.

## What is a Channel?

Channels are named pathways for routing messages in PubNub. They act as labels that connect publishers with subscribers.

### Key Characteristics

- **Dynamic Creation**: Channels exist when used - no pre-registration needed
- **Lightweight**: No additional cost for number of channels used
- **Unlimited**: Use as many channels as your application requires
- **Case-Sensitive**: `MyChannel` and `mychannel` are different

## Channel Naming Rules

### Valid Names

- **Length**: Up to 92 characters
- **Character Set**: UTF-8 compatible

### Invalid Characters (DO NOT USE)

| Character | Name |
|-----------|------|
| `,` | Comma |
| `:` | Colon |
| `*` | Asterisk |
| `/` | Slash |
| `\` | Backslash |
| ` ` | Space |
| `.` | Period (reserved for wildcards) |

### Naming Conventions

```javascript
// Good channel names
'chat-room-123'
'user_notifications_alice'
'sports-scores-nba'
'iot-sensor-device42'

// Bad channel names (will cause errors)
'chat room'      // space
'user:alice'     // colon
'sports/nba'     // slash
'alerts.*'       // wildcard (use only for subscriptions)
```

## Channel Design Patterns

### 1. Direct Messaging (1:1 Chat)

```javascript
// Convention: Sort user IDs alphabetically for consistent channel name
function getDirectChannelName(userId1, userId2) {
  const sorted = [userId1, userId2].sort();
  return `dm-${sorted[0]}-${sorted[1]}`;
}

// Results in: 'dm-alice-bob' (always the same regardless of who initiates)
```

### 2. Group Chat Rooms

```javascript
// Room-based channels
'room-project-alpha'
'room-social-lounge'
'room-support-team'

// With namespace prefix
'chat-group-12345'
```

### 3. User-Specific Channels

```javascript
// Personal notifications
`user-notifications-${userId}`

// User activity feed
`user-feed-${userId}`

// User presence channel
`user-status-${userId}`
```

### 4. Hierarchical Topics (for Wildcard Subscribe)

```javascript
// Sports hierarchy
'sports.football.scores'
'sports.basketball.news'
'sports.baseball.highlights'

// IoT sensor hierarchy
'iot.building1.floor2.temp'
'iot.building1.floor2.humidity'

// Subscribe to all with wildcard
pubnub.subscribe({ channels: ['sports.*'] });
```

### 5. Device/IoT Channels

```javascript
// Device telemetry
`device-telemetry-${deviceId}`

// Device commands
`device-commands-${deviceId}`

// Fleet-wide broadcasts
'devices-all-updates'
```

## Wildcard Subscriptions

> **Requires**: Stream Controller enabled in Admin Portal

### Pattern Rules

- Wildcard (`*`) must be at the **end** of the pattern
- Maximum **two dots** (three levels)
- Period (`.`) is the hierarchy delimiter

### Valid Patterns

```javascript
// One level
pubnub.subscribe({ channels: ['news.*'] });
// Matches: news.sports, news.weather, news.politics

// Two levels
pubnub.subscribe({ channels: ['iot.building1.*'] });
// Matches: iot.building1.temp, iot.building1.humidity
```

### Invalid Patterns

```javascript
// Wildcard not at end - INVALID
'news.*.sports'

// Too many levels - INVALID
'a.b.c.d.*'

// Wildcard at start - INVALID
'*.notifications'
```

## Channel Groups

> **Requires**: Stream Controller enabled in Admin Portal

Channel Groups allow subscribing to thousands of channels efficiently.

### Limits

| Resource | Limit |
|----------|-------|
| Channels per group | 2,000 |
| Groups per client | 10 |
| Total channels via groups | 20,000 |

### Creating and Managing Groups

```javascript
// Add channels to a group
await pubnub.channelGroups.addChannels({
  channelGroup: 'user-alice-feeds',
  channels: ['feed-news', 'feed-sports', 'feed-tech']
});

// Subscribe to the group
pubnub.subscribe({
  channelGroups: ['user-alice-feeds']
});

// Remove channels from group
await pubnub.channelGroups.removeChannels({
  channelGroup: 'user-alice-feeds',
  channels: ['feed-tech']
});

// List channels in group
const result = await pubnub.channelGroups.listChannels({
  channelGroup: 'user-alice-feeds'
});
console.log('Channels:', result.channels);

// Delete entire group
await pubnub.channelGroups.deleteChannelGroup({
  channelGroup: 'user-alice-feeds'
});
```

### Important Notes

- **Cannot publish to group**: Always publish to individual channels
- **Cannot use wildcards in groups**: Wildcard patterns not allowed
- **Group must have channels**: Subscribe to empty group returns 400 error

## Multiplexing

Subscribe to multiple named channels over a single connection.

```javascript
// Subscribe to up to 30 channels (recommended max)
pubnub.subscribe({
  channels: ['chat-room', 'notifications', 'presence-updates']
});
```

### When to Use What

| Scenario | Approach |
|----------|----------|
| < 30 channels | Multiplexing |
| 30-2,000 channels | Channel Groups |
| > 2,000 channels | Multiple Channel Groups |
| Hierarchical data | Wildcard Subscribe |

## Channel Access Control

By default, any client can publish/subscribe to any channel. Use Access Manager to restrict:

```javascript
// Server-side: Grant read access to specific channel
await pubnub.grant({
  channels: ['private-room-123'],
  authKeys: ['user-auth-token'],
  read: true,
  write: false,
  ttl: 60  // minutes
});
```

## Message Buffer (Short-Term Cache)

Each channel maintains a short-term buffer:

- Holds last **100 messages**
- Retention: up to **~20 minutes**
- Used for quick catch-up after brief disconnects
- High publish rates push old messages out faster

For longer storage, enable **Message Persistence** add-on.

## Best Practices

1. **Use descriptive, hierarchical names** when planning for wildcards
2. **Avoid special characters** - stick to alphanumeric, hyphens, underscores
3. **Keep names short** but meaningful (92 char limit)
4. **Document your channel naming scheme** for team consistency
5. **Use Channel Groups** for user-specific aggregations (feeds, notifications)
6. **Plan for scale** - structure channels for wildcard patterns if needed
