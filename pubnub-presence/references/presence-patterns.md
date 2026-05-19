<!-- canonical-for: PRESENCE_PATTERNS -->
<!-- used-by: -->

> **Cross-references:** [`new PubNub()` initialization, userId/UUID, `addListener` and pubnub.subscribe basics](../../pubnub-app-developer/references/sdk-patterns.md) (and [pub/sub patterns](../../pubnub-app-developer/references/publish-subscribe.md)). [Reconnect with backoff](../../pubnub-reliability/references/backoff-and-jitter.md), [dedup-on-merge](../../pubnub-reliability/references/dedup-on-merge.md), [multi-device sync](multi-device-sync.md), [App Context user metadata](../../pubnub-app-context/references/users.md).

# PubNub Presence Best Practices

## Best Practices Overview

### 1. Unique User Identification

**Always use a unique, persistent userId**:

```javascript
// Generate once and persist
function getUserId() {
  let userId = localStorage.getItem('pubnub_user_id');
  if (!userId) {
    userId = `user_${crypto.randomUUID()}`;
    localStorage.setItem('pubnub_user_id', userId);
  }
  return userId;
}

const pubnub = new PubNub({
  publishKey: 'pub-c-...',
  subscribeKey: 'sub-c-...',
  userId: getUserId()
});
```

**Why**: Without consistent IDs, presence events become unreliable and billing is affected.

### 2. Enable Event Engine

```javascript
const pubnub = new PubNub({
  publishKey: 'pub-c-...',
  subscribeKey: 'sub-c-...',
  userId: getUserId(),
  enableEventEngine: true  // Recommended for connection reliability
});
```

**Why**: Event Engine provides automatic reconnection, better error handling, and more predictable connection states.

### 3. Get Initial State on Connect

```javascript
pubnub.addListener({
  status: (statusEvent) => {
    if (statusEvent.category === 'PNConnectedCategory') {
      // Fetch current occupancy when connected
      getInitialOccupancy();
    }
  }
});

async function getInitialOccupancy() {
  try {
    const result = await pubnub.hereNow({
      channels: ['chat-room'],
      includeUUIDs: true,
      includeState: true
    });

    const channelData = result.channels['chat-room'];
    if (channelData) {
      updateUserList(channelData.occupants);
      updateOccupancyCount(channelData.occupancy);
    }
  } catch (error) {
    console.error('Error fetching initial presence:', error);
    updateOccupancyCount(0);
  }
}
```

**Why**: Presence events only tell you about changes. You need hereNow for the initial state.

### 4. Optimize hereNow Calls

```javascript
// Only request what you need
const result = await pubnub.hereNow({
  channels: ['chat-room'],
  includeUUIDs: false,  // Don't fetch UUIDs if you only need count
  includeState: false   // Don't fetch state if not needed
});

// More efficient for occupancy-only use cases
```

**Why**: Including UUIDs and state increases payload size and processing time.

### 5. Proper Cleanup

```javascript
// Always unsubscribe on page close for accurate presence
window.addEventListener('beforeunload', () => {
  pubnub.unsubscribeAll();
});

// React cleanup example
useEffect(() => {
  const subscription = channel.subscription({ receivePresenceEvents: true });
  subscription.subscribe();

  return () => {
    subscription.unsubscribe();  // Clean exit
  };
}, []);
```

**Why**: Without cleanup, users appear online until timeout (could be minutes).

## Scalable Presence Patterns

### High-Occupancy Channel Pattern

For channels with many users (1000+), use interval events:

```javascript
pubnub.addListener({
  presence: (event) => {
    if (event.action === 'interval') {
      // Batch updates instead of individual events
      console.log(`Occupancy: ${event.occupancy}`);

      // Process batch changes
      if (event.join) {
        event.join.forEach(uuid => addUserToList(uuid));
      }
      if (event.leave) {
        event.leave.forEach(uuid => removeUserFromList(uuid));
      }
    }
  }
});
```

### Presence Caching

```javascript
class PresenceCache {
  constructor(ttl = 30000) {  // 30 second cache
    this.cache = new Map();
    this.ttl = ttl;
  }

  async getOccupancy(channel, pubnub) {
    const cached = this.cache.get(channel);
    if (cached && Date.now() - cached.timestamp < this.ttl) {
      return cached.occupancy;
    }

    const result = await pubnub.hereNow({
      channels: [channel],
      includeUUIDs: false
    });

    const occupancy = result.channels[channel]?.occupancy || 0;
    this.cache.set(channel, {
      occupancy,
      timestamp: Date.now()
    });

    return occupancy;
  }
}
```

### Multi-Channel Presence

```javascript
// Fetch presence for multiple channels at once
const result = await pubnub.hereNow({
  channels: ['room-1', 'room-2', 'room-3'],
  includeUUIDs: false
});

// Process each channel
for (const [channel, data] of Object.entries(result.channels)) {
  console.log(`${channel}: ${data.occupancy} users`);
}
```

## User State Management

### Setting Custom State

```javascript
// Set state on subscribe
pubnub.subscribe({
  channels: ['chat-room'],
  withPresence: true,
  state: {
    status: 'available',
    nickname: 'Alice',
    avatar: 'https://example.com/avatar.png'
  }
});

// Update state dynamically
async function updateStatus(newStatus) {
  await pubnub.setState({
    channels: ['chat-room'],
    state: {
      status: newStatus,
      lastUpdated: Date.now()
    }
  });
}

// Usage
updateStatus('away');
updateStatus('busy');
updateStatus('available');
```

### Reading User State

```javascript
// Get specific user's state
const result = await pubnub.getState({
  channels: ['chat-room'],
  uuid: 'other-user-id'
});

console.log('User state:', result.channels['chat-room']);
```

## Connection Management

### Detecting Connection Status

```javascript
let isOnline = false;

pubnub.addListener({
  status: (statusEvent) => {
    switch (statusEvent.category) {
      case 'PNConnectedCategory':
        isOnline = true;
        showOnlineIndicator();
        break;

      case 'PNReconnectedCategory':
        isOnline = true;
        showOnlineIndicator();
        // Refresh presence state after reconnect
        refreshPresenceState();
        break;

      case 'PNDisconnectedCategory':
        isOnline = false;
        showOfflineIndicator('Reconnecting...');
        break;

      case 'PNNetworkDownCategory':
        isOnline = false;
        showOfflineIndicator('No network connection');
        break;
    }
  }
});
```

### Synchronizing Multiple Devices

```javascript
// Use consistent userId across devices
const userId = authenticatedUser.id;  // From your auth system

const pubnub = new PubNub({
  publishKey: 'pub-c-...',
  subscribeKey: 'sub-c-...',
  userId: userId,  // Same ID on all devices
  // Optionally add device identifier to state
});

pubnub.subscribe({
  channels: ['user-presence'],
  withPresence: true,
  state: {
    deviceType: detectDeviceType(),  // 'mobile', 'desktop', 'tablet'
    deviceId: getDeviceId()
  }
});
```

## Performance Guidelines

### When to Use Presence

| Use Case | Recommendation |
|----------|----------------|
| Chat room (< 100 users) | Full presence with user list |
| Chat room (100-1000 users) | Occupancy count only |
| Chat room (1000+ users) | Consider disabling or sampling |
| Gaming lobby | Full presence for matchmaking |
| IoT device status | Full presence with state |
| Live event (10K+ users) | Disable individual presence, use aggregated counts |

### Optimizing for Scale

```javascript
// For high-scale: Use announce max
// Configure in Admin Portal: Presence > Announce Max

// When occupancy exceeds announce max, you get interval events
// instead of individual join/leave events

// Handle both patterns:
presence: (event) => {
  if (event.action === 'interval') {
    // High occupancy: batch updates
    processIntervalEvent(event);
  } else {
    // Normal: individual events
    processIndividualEvent(event);
  }
}
```

### Reducing Presence Traffic

```javascript
// Only subscribe to presence where needed
pubnub.subscribe({
  channels: ['chat-room', 'notifications'],
  channelsWithPresence: ['chat-room']  // Presence only for chat
});

// Use presence selectively in your UI
// Don't poll hereNow - rely on events after initial fetch
```

## Summary Checklist

- [ ] Enable Presence in Admin Portal
- [ ] Configure channel rules in Presence Management
- [ ] Use persistent, unique userId
- [ ] Enable Event Engine for reliability
- [ ] Fetch initial state with hereNow on connect
- [ ] Handle all presence event types (join, leave, timeout, state-change, interval)
- [ ] Implement proper cleanup on page unload
- [ ] Optimize hereNow calls (only request needed fields)
- [ ] Consider announce max for high-occupancy channels
- [ ] Cache presence data to reduce API calls
