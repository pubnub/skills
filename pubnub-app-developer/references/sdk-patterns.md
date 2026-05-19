<!-- canonical-for: SDK_INITIALIZATION -->
<!-- canonical-for: USERID_UUID -->
<!-- used-by: pubnub-keyset-management, pubnub-app-context, pubnub-history, pubnub-presence, pubnub-reliability, pubnub-observability, pubnub-events-and-actions, pubnub-illuminate, pubnub-chat -->

> **Cross-references:** For [Secret Key, publish key, subscribe key](../../pubnub-keyset-management/references/keysets-and-environments.md) handling see the canonical owner. For [Access Manager](../../pubnub-security/references/access-manager.md) tokens, [AES-256 message encryption / cipherKey](../../pubnub-security/references/encryption.md), [Presence event semantics](../../pubnub-presence/references/presence-events.md), [PNNetworkDownCategory and dropped-connection categories](../../pubnub-presence/references/dropped-connections.md), [Message action and reaction APIs](../../pubnub-chat/references/message-actions.md), and [Event Listener (Events & Actions)](../../pubnub-events-and-actions/references/event-types.md) see the canonical owners.

# PubNub SDK Initialization and Configuration

## SDK Initialization

### JavaScript/TypeScript

```javascript
import PubNub from 'pubnub';

const pubnub = new PubNub({
  publishKey: 'pub-c-...',
  subscribeKey: 'sub-c-...',
  userId: 'unique-user-id',           // REQUIRED - persistent identifier
  // Optional configuration
  ssl: true,                           // Default: true (use HTTPS)
  authKey: 'auth-token',              // For Access Manager authentication
  cipherKey: 'my-cipher-key',         // For AES-256 message encryption
  logVerbosity: false,                // Enable debug logging
  enableEventEngine: true             // Recommended for better connection management
});
```

### Python

```python
from pubnub.pnconfiguration import PNConfiguration
from pubnub.pubnub import PubNub

pnconfig = PNConfiguration()
pnconfig.publish_key = 'pub-c-...'
pnconfig.subscribe_key = 'sub-c-...'
pnconfig.uuid = 'unique-user-id'      # REQUIRED
pnconfig.ssl = True                    # Default: True
pnconfig.auth_key = 'auth-token'      # For Access Manager
pnconfig.cipher_key = 'my-cipher-key' # For encryption

pubnub = PubNub(pnconfig)
```

### Swift (iOS)

```swift
import PubNub

let config = PubNubConfiguration(
    publishKey: "pub-c-...",
    subscribeKey: "sub-c-...",
    userId: "unique-user-id"
)
config.authKey = "auth-token"        // For Access Manager
config.cipherKey = "my-cipher-key"   // For encryption

let pubnub = PubNub(configuration: config)
```

### Kotlin (Android)

```kotlin
import com.pubnub.api.PNConfiguration
import com.pubnub.api.PubNub
import com.pubnub.api.UserId

val config = PNConfiguration(UserId("unique-user-id")).apply {
    publishKey = "pub-c-..."
    subscribeKey = "sub-c-..."
    authKey = "auth-token"          // For Access Manager
    cipherKey = "my-cipher-key"     // For encryption
}

val pubnub = PubNub(config)
```

### Java

```java
import com.pubnub.api.PNConfiguration;
import com.pubnub.api.PubNub;

PNConfiguration config = new PNConfiguration();
config.setPublishKey("pub-c-...");
config.setSubscribeKey("sub-c-...");
config.setUuid("unique-user-id");    // REQUIRED
config.setAuthKey("auth-token");     // For Access Manager
config.setCipherKey("my-cipher-key"); // For encryption

PubNub pubnub = new PubNub(config);
```

## userId (UUID) Requirements

### Critical Rules

1. **Always Required**: SDK will error without a userId
2. **Must Be Unique**: Each user/device needs a distinct identifier
3. **Must Be Persistent**: Reuse the same userId across sessions
4. **Max Length**: 64 characters

### Bad Practices

```javascript
// DON'T: Generate random UUID on each session
userId: crypto.randomUUID()  // Creates billing/presence issues

// DON'T: Use undefined or empty
userId: ''  // Will cause errors
userId: undefined  // Will cause errors
```

### Good Practices

```javascript
// DO: Generate once and persist
function getUserId() {
  let userId = localStorage.getItem('pubnub_user_id');
  if (!userId) {
    userId = `user_${crypto.randomUUID()}`;
    localStorage.setItem('pubnub_user_id', userId);
  }
  return userId;
}

// DO: Use authenticated user ID from your system
userId: authenticatedUser.id

// DO: Use device identifier for IoT
userId: `device_${deviceSerialNumber}`
```

## Event Listeners

### JavaScript Listener Pattern

```javascript
pubnub.addListener({
  // Receive messages
  message: (event) => {
    console.log('Message:', event.message);
    console.log('Channel:', event.channel);
    console.log('Publisher:', event.publisher);
    console.log('Timetoken:', event.timetoken);
  },

  // Connection status
  status: (statusEvent) => {
    console.log('Status category:', statusEvent.category);
    console.log('Affected channels:', statusEvent.affectedChannels);

    switch (statusEvent.category) {
      case 'PNConnectedCategory':
        console.log('Connected to PubNub');
        break;
      case 'PNReconnectedCategory':
        console.log('Reconnected after disconnect');
        break;
      case 'PNDisconnectedCategory':
        console.warn('Disconnected - will auto-reconnect');
        break;
      case 'PNNetworkDownCategory':
        console.error('Network is down');
        break;
      case 'PNNetworkUpCategory':
        console.log('Network restored');
        break;
      case 'PNAccessDeniedCategory':
        console.error('Access denied - check auth token');
        break;
    }
  },

  // Presence events (if subscribed with presence)
  presence: (presenceEvent) => {
    console.log('Action:', presenceEvent.action);  // join, leave, timeout
    console.log('UUID:', presenceEvent.uuid);
    console.log('Occupancy:', presenceEvent.occupancy);
  },

  // Signal events
  signal: (signalEvent) => {
    console.log('Signal:', signalEvent.message);
  },

  // Message actions (reactions, etc.)
  messageAction: (actionEvent) => {
    console.log('Action type:', actionEvent.data.type);
    console.log('Action value:', actionEvent.data.value);
  }
});
```

### Python Listener Pattern

```python
from pubnub.callbacks import SubscribeCallback
from pubnub.enums import PNStatusCategory

class MySubscribeCallback(SubscribeCallback):
    def message(self, pubnub, message):
        print(f'Message: {message.message}')
        print(f'Channel: {message.channel}')
        print(f'Publisher: {message.publisher}')

    def status(self, pubnub, status):
        if status.category == PNStatusCategory.PNConnectedCategory:
            print('Connected')
        elif status.category == PNStatusCategory.PNReconnectedCategory:
            print('Reconnected')
        elif status.category == PNStatusCategory.PNDisconnectedCategory:
            print('Disconnected')

    def presence(self, pubnub, presence):
        print(f'Action: {presence.event}')
        print(f'UUID: {presence.uuid}')

pubnub.add_listener(MySubscribeCallback())
```

## Common Configuration Options

| Option | Description | Default |
|--------|-------------|---------|
| `publishKey` | Key for publishing messages | Required |
| `subscribeKey` | Key for subscribing to channels | Required |
| `userId` | Unique client identifier | Required |
| `ssl` | Use HTTPS connections | `true` |
| `authKey` | Access Manager authentication token | None |
| `cipherKey` | AES-256 encryption key | None |
| `logVerbosity` | Enable debug logging | `false` |
| `enableEventEngine` | Modern connection management | Varies |
| `heartbeatInterval` | Presence heartbeat interval (seconds) | 300 |
| `presenceTimeout` | Presence timeout (seconds) | 300 |

## Error Handling Patterns

### Publish with Error Handling

```javascript
async function safePublish(channel, message) {
  try {
    const result = await pubnub.publish({ channel, message });
    console.log('Published with timetoken:', result.timetoken);
    return result;
  } catch (error) {
    console.error('Publish failed:', error.status);

    if (error.status?.category === 'PNAccessDeniedCategory') {
      // Refresh auth token and retry
      await refreshAuthToken();
      return pubnub.publish({ channel, message });
    }
    throw error;
  }
}
```

### Connection Retry Logic

```javascript
let reconnectAttempts = 0;
const MAX_RECONNECT_ATTEMPTS = 5;

pubnub.addListener({
  status: (statusEvent) => {
    if (statusEvent.category === 'PNDisconnectedCategory') {
      reconnectAttempts++;

      if (reconnectAttempts >= MAX_RECONNECT_ATTEMPTS) {
        console.error('Max reconnection attempts reached');
        // Notify user, show offline UI
      }
    } else if (statusEvent.category === 'PNConnectedCategory' ||
               statusEvent.category === 'PNReconnectedCategory') {
      reconnectAttempts = 0;
    }
  }
});
```

## Cleanup and Lifecycle

### React Component Example

```javascript
import { useEffect, useRef } from 'react';
import PubNub from 'pubnub';

function ChatComponent({ userId, channel }) {
  const pubnubRef = useRef(null);

  useEffect(() => {
    // Initialize
    const pubnub = new PubNub({
      publishKey: process.env.REACT_APP_PUBNUB_PUB_KEY,
      subscribeKey: process.env.REACT_APP_PUBNUB_SUB_KEY,
      userId
    });
    pubnubRef.current = pubnub;

    // Add listener
    const listener = {
      message: (event) => {
        // Handle messages
      }
    };
    pubnub.addListener(listener);

    // Subscribe
    pubnub.subscribe({ channels: [channel] });

    // Cleanup on unmount
    return () => {
      pubnub.removeListener(listener);
      pubnub.unsubscribeAll();
    };
  }, [userId, channel]);

  return (/* ... */);
}
```

### Page Unload Cleanup

```javascript
window.addEventListener('beforeunload', () => {
  if (pubnub) {
    pubnub.unsubscribeAll();
  }
});
```

## Best Practices Summary

1. **Initialize once**: Create a single PubNub instance per user session
2. **Add listeners before subscribing**: Ensure you don't miss events
3. **Handle all status categories**: Provide good UX during disconnects
4. **Clean up on unmount**: Unsubscribe to prevent memory leaks
5. **Never expose Secret Key**: Only use on server-side code
6. **Persist userId**: Avoid random IDs that change per session
7. **Use try/catch**: Handle publish/subscribe errors gracefully
