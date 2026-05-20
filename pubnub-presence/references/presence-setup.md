<!-- canonical-for: PRESENCE_SETUP -->
<!-- used-by: -->

> **Cross-references:** Presence add-on enablement is part of [keyset configuration](../../pubnub-keyset-management/references/keysets-and-environments.md). For [SDK initialization](../../pubnub-app-developer/references/sdk-patterns.md) options including heartbeat and the [`pubnub.subscribe`](../../pubnub-app-developer/references/publish-subscribe.md) call shape see the canonical owners. Presence on restricted channels needs [Access Manager grants](../../pubnub-security/references/access-manager.md).

# PubNub Presence Setup Guide

## Enabling Presence

### Admin Portal Configuration

**Step 1: Enable Presence Add-on**
1. Log in to PubNub Admin Portal
2. Select your Application and Keyset
3. Navigate to "Key Add-ons" or Keyset settings
4. Enable the **Presence** add-on

**Step 2: Configure Presence Mode**
Choose between:
- **All channels**: Presence tracked on every channel (high cost, use with caution)
- **Selected channels only (RECOMMENDED)**: Control which channels track presence

**Step 3: Configure Channel Rules (if using "Selected channels only")**
1. Go to **Presence Management** section
2. Add channel patterns that should track presence
3. Examples:
   - `chat-room-*` - All chat rooms
   - `lobby` - Specific channel
   - `game-*` - All game channels

> **Important**: Without configuring channel rules in Presence Management, presence will NOT work even if enabled in code.

## SDK Configuration

### Basic Presence Subscription

```javascript
const pubnub = new PubNub({
  publishKey: 'pub-c-...',
  subscribeKey: 'sub-c-...',
  userId: 'unique-user-id'
});

// Subscribe with presence
pubnub.subscribe({
  channels: ['chat-room'],
  withPresence: true
});
```

### Using Event Engine (Recommended)

```javascript
const pubnub = new PubNub({
  publishKey: 'pub-c-...',
  subscribeKey: 'sub-c-...',
  userId: 'unique-user-id',
  enableEventEngine: true  // Better connection management
});

// Modern channel subscription API
const channel = pubnub.channel('chat-room');
const subscription = channel.subscription({
  receivePresenceEvents: true
});

subscription.onPresence = (event) => {
  console.log('Presence event:', event);
};

subscription.subscribe();
```

## Heartbeat Configuration

### Understanding Heartbeats

- **Heartbeat Interval**: How often client sends "I'm alive" signal
- **Presence Timeout**: How long before user is considered offline after missed heartbeats
- Timeout should be >= 2 * heartbeat interval

### Configuration Options

```javascript
const pubnub = new PubNub({
  publishKey: 'pub-c-...',
  subscribeKey: 'sub-c-...',
  userId: 'unique-user-id',
  heartbeatInterval: 60,    // Send heartbeat every 60 seconds
  presenceTimeout: 300      // Consider offline after 300 seconds
});
```

### Recommended Settings by Use Case

| Use Case | Heartbeat Interval | Presence Timeout |
|----------|-------------------|------------------|
| Real-time chat | 30 | 120 |
| Gaming | 15 | 60 |
| IoT monitoring | 60 | 300 |
| Social presence | 60 | 300 |
| Battery-sensitive mobile | 120 | 600 |

## Presence Channels

When you subscribe with presence, PubNub creates a companion presence channel:

```
Original channel:  chat-room
Presence channel:  chat-room-pnpres
```

You don't subscribe to `-pnpres` directly - it's handled automatically.

## User State with Presence

Associate metadata with a user's presence:

```javascript
// Set state when subscribing
pubnub.subscribe({
  channels: ['chat-room'],
  withPresence: true,
  state: {
    status: 'available',
    mood: 'happy',
    device: 'mobile'
  }
});

// Update state later
await pubnub.setState({
  channels: ['chat-room'],
  state: {
    status: 'busy',
    mood: 'focused'
  }
});

// Get user's current state
const result = await pubnub.getState({
  channels: ['chat-room'],
  uuid: 'other-user-id'
});
```

## Verifying Presence Setup

### Test Configuration

```javascript
// 1. Subscribe with presence
pubnub.subscribe({
  channels: ['test-presence'],
  withPresence: true
});

// 2. Check hereNow after subscribing
setTimeout(async () => {
  const result = await pubnub.hereNow({
    channels: ['test-presence'],
    includeUUIDs: true
  });
  console.log('Occupancy:', result.channels['test-presence']?.occupancy);
  console.log('UUIDs:', result.channels['test-presence']?.occupants);
}, 2000);
```

### Common Issues

| Problem | Solution |
|---------|----------|
| No presence events | Check channel rules in Presence Management |
| Occupancy always 0 | Ensure Presence add-on is enabled |
| Events delayed | Check heartbeat/timeout configuration |
| Duplicate join events | Ensure consistent userId across sessions |

## Python Setup

```python
from pubnub.pnconfiguration import PNConfiguration
from pubnub.pubnub import PubNub
from pubnub.callbacks import SubscribeCallback

pnconfig = PNConfiguration()
pnconfig.subscribe_key = 'sub-c-...'
pnconfig.publish_key = 'pub-c-...'
pnconfig.uuid = 'unique-user-id'
pnconfig.heartbeat = 60
pnconfig.presence_timeout = 300

pubnub = PubNub(pnconfig)

class PresenceCallback(SubscribeCallback):
    def presence(self, pubnub, presence):
        print(f'Action: {presence.event}')
        print(f'UUID: {presence.uuid}')
        print(f'Occupancy: {presence.occupancy}')

pubnub.add_listener(PresenceCallback())
pubnub.subscribe().channels(['chat-room']).with_presence().execute()
```

## Presence with Access Manager

When using Access Manager, grant presence permissions:

```javascript
// Server-side grant
await pubnub.grant({
  channels: ['chat-room'],
  authKeys: ['user-auth-token'],
  read: true,
  write: true,
  manage: false,
  ttl: 60
});

// Grant for presence channel specifically
await pubnub.grant({
  channels: ['chat-room-pnpres'],
  authKeys: ['user-auth-token'],
  read: true,
  ttl: 60
});
```
