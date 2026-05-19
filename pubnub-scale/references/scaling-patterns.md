<!-- canonical-for: CHANNEL_GROUPS, WILDCARD_SUBSCRIBE -->
<!-- used-by: -->

> **Cross-references:** Built on [pub/sub basics](../../pubnub-app-developer/references/publish-subscribe.md), [SDK initialization (`new PubNub(`, `userId`/UUID)](../../pubnub-app-developer/references/sdk-patterns.md), and [channel naming](../../pubnub-app-developer/references/channels.md). For [history/Message Persistence](../../pubnub-history/references/pagination-and-ordering.md) see the canonical owner. For [presence on grouped/wildcard subscribes (`withPresence`, presence event semantics)](../../pubnub-presence/references/presence-events.md) (and [setup](../../pubnub-presence/references/presence-setup.md)) see `pubnub-presence`. [Access Manager grants](../../pubnub-security/references/access-manager.md) for groups apply at the group level. [Custom origin / vanity domain](../../pubnub-keyset-management/references/custom-origin.md) is owned by `pubnub-keyset-management`. Watch fan-out impact on [usage metrics / transaction count](../../pubnub-observability/references/usage-metrics.md).

# PubNub Scaling Patterns

## Overview

PubNub is designed for massive scale:
- **Millions** of concurrent connections
- **Billions** of messages per day
- **15+ global data centers** (Points of Presence)
- **Sub-100ms** delivery target

## Subscription Strategies

### When to Use What

| Channels Needed | Strategy | Max Channels |
|-----------------|----------|--------------|
| 1-30 | Multiplexing | 30 recommended |
| 30-2,000 | Single Channel Group | 2,000 |
| 2,000-20,000 | Multiple Channel Groups | 10 groups × 2,000 |
| Hierarchical | Wildcard Subscribe | Unlimited matching |

## Channel Multiplexing

Subscribe to multiple named channels over single connection.

```javascript
// Subscribe to multiple channels (up to 30 recommended)
pubnub.subscribe({
  channels: ['chat-room-1', 'chat-room-2', 'notifications']
});
```

**Best for**: Small, known set of channels per client.

## Channel Groups

> **Requires**: Stream Controller enabled in Admin Portal

### Setup

```javascript
// Add channels to a group
await pubnub.channelGroups.addChannels({
  channelGroup: 'user-alice-feeds',
  channels: [
    'feed-news',
    'feed-sports',
    'feed-tech',
    'user-notifications-alice'
  ]
});

// Subscribe to the group
pubnub.subscribe({
  channelGroups: ['user-alice-feeds']
});
```

### Managing Channel Groups

```javascript
// List channels in group
const result = await pubnub.channelGroups.listChannels({
  channelGroup: 'user-alice-feeds'
});
console.log('Channels:', result.channels);

// Remove channels from group
await pubnub.channelGroups.removeChannels({
  channelGroup: 'user-alice-feeds',
  channels: ['feed-tech']
});

// Delete entire group
await pubnub.channelGroups.deleteChannelGroup({
  channelGroup: 'user-alice-feeds'
});
```

### Limits

| Resource | Limit |
|----------|-------|
| Channels per group | 2,000 (configurable) |
| Groups per client | 10 |
| Total channels | 20,000 per client |

### User Feed Pattern

```javascript
// Server-side: Create personalized feed group
async function createUserFeedGroup(userId, subscriptions) {
  const groupName = `user-${userId}-feeds`;

  // Add user's subscribed topics
  await pubnub.channelGroups.addChannels({
    channelGroup: groupName,
    channels: subscriptions.map(topic => `feed-${topic}`)
  });

  // Add user's personal channels
  await pubnub.channelGroups.addChannels({
    channelGroup: groupName,
    channels: [
      `user-notifications-${userId}`,
      `user-dm-${userId}`
    ]
  });

  return groupName;
}

// Client-side: Subscribe to personal feed
const feedGroup = `user-${currentUserId}-feeds`;
pubnub.subscribe({
  channelGroups: [feedGroup]
});
```

## Wildcard Subscriptions

> **Requires**: Stream Controller enabled in Admin Portal

### Pattern Rules

- Wildcard (`*`) must be at the **end**
- Maximum **2 dots** (3 levels)
- Period (`.`) is the hierarchy delimiter

### Valid Patterns

```javascript
// One level
pubnub.subscribe({ channels: ['sports.*'] });
// Matches: sports.football, sports.basketball, sports.baseball

// Two levels
pubnub.subscribe({ channels: ['iot.building1.*'] });
// Matches: iot.building1.temp, iot.building1.humidity

// Specific namespace
pubnub.subscribe({ channels: ['stocks.nasdaq.*'] });
// Matches: stocks.nasdaq.aapl, stocks.nasdaq.goog
```

### Invalid Patterns

```javascript
// Wildcard not at end - INVALID
'sports.*.scores'

// Wildcard at start - INVALID
'*.notifications'

// Too many levels - INVALID
'a.b.c.d.*'
```

### IoT Sensor Pattern

```javascript
// Publish sensor data to specific channel
await pubnub.publish({
  channel: 'sensors.floor1.room101.temperature',
  message: { value: 72.5, unit: 'F', timestamp: Date.now() }
});

// Subscribe to all sensors on floor 1
pubnub.subscribe({
  channels: ['sensors.floor1.*']
});

// Listener receives from all matching channels
pubnub.addListener({
  message: (event) => {
    console.log('Channel:', event.channel);  // sensors.floor1.room101.temperature
    console.log('Data:', event.message);
  }
});
```

## High Concurrency Guidelines

### Contact PubNub When

- **1,000+ concurrent users** regularly - discuss Pro plan
- **10,000+ concurrent users** for events - submit Virtual Event Form
- Need **architecture review** for scale
- Require **live event support**

### Chat Room Scaling

```javascript
// For < 10,000 users in a room: Single channel works well
'chat-room-main'

// For > 10,000 users: Consider sharding
function getShardedChannel(roomId, userId) {
  const shardCount = 10;
  const shardId = hashCode(userId) % shardCount;
  return `chat-room-${roomId}-shard-${shardId}`;
}

// Users subscribe to their shard
const myChannel = getShardedChannel('main', currentUserId);
pubnub.subscribe({ channels: [myChannel] });

// Broadcast messages to all shards
async function broadcastToRoom(roomId, message) {
  const shardCount = 10;
  const publishes = [];
  for (let i = 0; i < shardCount; i++) {
    publishes.push(pubnub.publish({
      channel: `chat-room-${roomId}-shard-${i}`,
      message
    }));
  }
  await Promise.all(publishes);
}
```

### Presence at Scale

```javascript
// For high-occupancy channels, use interval events
// Configure "Announce Max" in Admin Portal

pubnub.addListener({
  presence: (event) => {
    if (event.action === 'interval') {
      // Batch updates for high occupancy
      console.log('Occupancy:', event.occupancy);
      if (event.join) event.join.forEach(handleJoin);
      if (event.leave) event.leave.forEach(handleLeave);
    } else {
      // Individual events for normal occupancy
      handlePresenceEvent(event);
    }
  }
});
```

## Connection Management

### Browser Connection Limits

Browsers limit connections per hostname (Chrome: 6).

```javascript
// PubNub uses efficient connection pooling
// One subscribe connection + one for publishes/API calls

// For multiple PubNub instances (rare), consider custom origin
// Contact PubNub support for custom origin: yoursubdomain.pubnubapi.com
```

### Keep-Alive Configuration

```javascript
// PubNub SDKs maintain long-lived connections
// Default: 1-hour keep-alive with 5-minute pings

// For battery-sensitive mobile apps
const pubnub = new PubNub({
  subscribeKey: 'sub-c-...',
  userId: 'user-123',
  heartbeatInterval: 120,  // Less frequent heartbeats
  presenceTimeout: 600     // Longer timeout
});
```

## Monitoring Scale

### PubNub Tools

- **Speed-o-Meter**: https://www.pubnub.com/developers/speed/
- **Status Page**: https://status.pubnub.com/
- **Admin Portal**: Usage metrics and analytics

### SDK Latency Metrics

```javascript
pubnub.addListener({
  status: (statusEvent) => {
    if (statusEvent.operation === 'PNPublishOperation') {
      console.log('Publish latency info:', statusEvent);
    }
  }
});
```
