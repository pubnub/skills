<!-- canonical-for: PRESENCE -->
<!-- used-by: pubnub-keyset-management, pubnub-reliability, pubnub-observability, pubnub-events-and-actions, pubnub-app-context -->

> **Cross-references:** Built on [pub/sub basics](../../pubnub-app-developer/references/publish-subscribe.md). Per-user identification via [userId/UUID](../../pubnub-app-developer/references/sdk-patterns.md). [Access Manager grants](../../pubnub-security/references/access-manager.md) protect presence data on restricted channels. Presence events also flow into [Events & Actions Channels source](../../pubnub-events-and-actions/references/event-types.md).

# PubNub Presence Events

## Event Types

| Event | Description | Triggered When |
|-------|-------------|----------------|
| `join` | User came online | Client subscribes to channel |
| `leave` | User went offline gracefully | Client unsubscribes or calls leave |
| `timeout` | User went offline ungracefully | No heartbeat received within timeout |
| `state-change` | User's state metadata changed | Client calls setState |
| `interval` | Periodic occupancy summary | High-occupancy channels (announce max) |

## Handling Presence Events

### JavaScript Listener

```javascript
pubnub.addListener({
  presence: (event) => {
    const { action, uuid, channel, occupancy, timestamp, state } = event;

    switch (action) {
      case 'join':
        console.log(`${uuid} joined ${channel}`);
        console.log(`Occupancy: ${occupancy}`);
        if (state) {
          console.log(`State: ${JSON.stringify(state)}`);
        }
        // Update UI: add user to online list
        addUserToList(uuid, state);
        break;

      case 'leave':
        console.log(`${uuid} left ${channel}`);
        console.log(`Occupancy: ${occupancy}`);
        // Update UI: remove user from online list
        removeUserFromList(uuid);
        break;

      case 'timeout':
        console.log(`${uuid} timed out from ${channel}`);
        // User disconnected without proper unsubscribe
        // Handle same as leave
        removeUserFromList(uuid);
        break;

      case 'state-change':
        console.log(`${uuid} updated state in ${channel}`);
        console.log(`New state: ${JSON.stringify(state)}`);
        // Update UI: refresh user info
        updateUserState(uuid, state);
        break;

      case 'interval':
        // Periodic summary for high-occupancy channels
        console.log(`Interval event - Occupancy: ${occupancy}`);
        if (event.join) {
          event.join.forEach(uuid => addUserToList(uuid));
        }
        if (event.leave) {
          event.leave.forEach(uuid => removeUserFromList(uuid));
        }
        break;
    }
  }
});
```

### Python Listener

```python
from pubnub.callbacks import SubscribeCallback

class PresenceCallback(SubscribeCallback):
    def presence(self, pubnub, presence):
        action = presence.event
        uuid = presence.uuid
        channel = presence.channel
        occupancy = presence.occupancy

        if action == 'join':
            print(f'{uuid} joined {channel}')
            print(f'Occupancy: {occupancy}')
            if presence.state:
                print(f'State: {presence.state}')

        elif action == 'leave':
            print(f'{uuid} left {channel}')

        elif action == 'timeout':
            print(f'{uuid} timed out')

        elif action == 'state-change':
            print(f'{uuid} changed state: {presence.state}')
```

## Event Object Properties

### Join Event

```javascript
{
  action: 'join',
  uuid: 'user-123',           // Who joined
  channel: 'chat-room',       // Channel name
  occupancy: 5,               // Total users now in channel
  timestamp: 1704067200,      // Unix timestamp
  state: {                    // User's state (if set)
    status: 'available',
    device: 'mobile'
  },
  subscription: 'chat-room',  // Subscription pattern
  timetoken: '17040672000000000'
}
```

### Leave Event

```javascript
{
  action: 'leave',
  uuid: 'user-123',
  channel: 'chat-room',
  occupancy: 4,               // Updated count after leave
  timestamp: 1704067260
}
```

### Timeout Event

```javascript
{
  action: 'timeout',
  uuid: 'user-123',
  channel: 'chat-room',
  occupancy: 4,
  timestamp: 1704067500       // When timeout was detected
}
```

### State-Change Event

```javascript
{
  action: 'state-change',
  uuid: 'user-123',
  channel: 'chat-room',
  state: {
    status: 'away',
    lastSeen: 1704067200
  },
  timestamp: 1704067300
}
```

### Interval Event (High Occupancy)

```javascript
{
  action: 'interval',
  channel: 'chat-room',
  occupancy: 150,
  timestamp: 1704067200,
  join: ['user-45', 'user-46'],     // Users who joined since last interval
  leave: ['user-10'],               // Users who left since last interval
  timeout: ['user-5']               // Users who timed out since last interval
}
```

## Building a User List

### Complete Implementation

```javascript
class PresenceManager {
  constructor(pubnub, channel) {
    this.pubnub = pubnub;
    this.channel = channel;
    this.users = new Map();  // uuid -> user info
    this.occupancy = 0;

    this.setupListener();
  }

  setupListener() {
    this.pubnub.addListener({
      presence: (event) => {
        if (event.channel !== this.channel) return;

        this.occupancy = event.occupancy;

        switch (event.action) {
          case 'join':
            this.users.set(event.uuid, {
              uuid: event.uuid,
              state: event.state || {},
              joinedAt: event.timestamp
            });
            this.onUserJoined(event.uuid, event.state);
            break;

          case 'leave':
          case 'timeout':
            this.users.delete(event.uuid);
            this.onUserLeft(event.uuid, event.action);
            break;

          case 'state-change':
            const user = this.users.get(event.uuid);
            if (user) {
              user.state = event.state;
              this.onUserStateChanged(event.uuid, event.state);
            }
            break;

          case 'interval':
            if (event.join) {
              event.join.forEach(uuid => {
                this.users.set(uuid, { uuid, state: {}, joinedAt: Date.now() });
              });
            }
            if (event.leave) {
              event.leave.forEach(uuid => this.users.delete(uuid));
            }
            if (event.timeout) {
              event.timeout.forEach(uuid => this.users.delete(uuid));
            }
            this.onIntervalUpdate();
            break;
        }

        this.onOccupancyChanged(this.occupancy);
      }
    });
  }

  async initialize() {
    // Fetch initial presence state
    const result = await this.pubnub.hereNow({
      channels: [this.channel],
      includeUUIDs: true,
      includeState: true
    });

    const channelData = result.channels[this.channel];
    if (channelData) {
      this.occupancy = channelData.occupancy;
      channelData.occupants.forEach(occupant => {
        this.users.set(occupant.uuid, {
          uuid: occupant.uuid,
          state: occupant.state || {},
          joinedAt: Date.now()
        });
      });
    }

    return this.users;
  }

  // Override these methods to react to changes
  onUserJoined(uuid, state) {}
  onUserLeft(uuid, reason) {}
  onUserStateChanged(uuid, state) {}
  onOccupancyChanged(count) {}
  onIntervalUpdate() {}

  getUsers() {
    return Array.from(this.users.values());
  }

  getOccupancy() {
    return this.occupancy;
  }

  isUserOnline(uuid) {
    return this.users.has(uuid);
  }
}

// Usage
const presence = new PresenceManager(pubnub, 'chat-room');
presence.onUserJoined = (uuid, state) => {
  console.log(`User ${uuid} joined with state:`, state);
};
presence.onOccupancyChanged = (count) => {
  document.getElementById('user-count').textContent = count;
};

// Initialize with current state
await presence.initialize();

// Subscribe to get live updates
pubnub.subscribe({
  channels: ['chat-room'],
  withPresence: true
});
```

## Handling Edge Cases

### Same User Multiple Devices

```javascript
// User might join from multiple devices
presence: (event) => {
  if (event.action === 'join') {
    // Check if user is already tracked
    const existing = users.get(event.uuid);
    if (existing) {
      // Increment device count instead of adding duplicate
      existing.deviceCount = (existing.deviceCount || 1) + 1;
    } else {
      users.set(event.uuid, {
        uuid: event.uuid,
        deviceCount: 1,
        state: event.state
      });
    }
  }
}
```

### Distinguish Leave vs Timeout

```javascript
presence: (event) => {
  if (event.action === 'leave') {
    // Graceful disconnect - user chose to leave
    showNotification(`${event.uuid} left the room`);
  } else if (event.action === 'timeout') {
    // Ungraceful disconnect - connection lost
    showNotification(`${event.uuid} disconnected`);
    // Optionally show "reconnecting" status briefly
    // as user might come back
  }
}
```
