<!-- canonical-for: PUBLISH_SUBSCRIBE_BASICS -->
<!-- used-by: pubnub-keyset-management, pubnub-app-context, pubnub-history, pubnub-reliability, pubnub-observability, pubnub-illuminate -->

> **Cross-references:** For [withPresence and presence event](../../pubnub-presence/references/presence-events.md) details and [PNNetworkDownCategory dropped-connection](../../pubnub-presence/references/dropped-connections.md) handling see the canonical owners. For [Message Persistence and fetchMessages](../../pubnub-history/references/pagination-and-ordering.md) see the History canonical owner.

# PubNub Publish/Subscribe Patterns

## Core Concepts

Publish/Subscribe (Pub/Sub) is PubNub's fundamental messaging pattern for real-time communication.

### Key Components

| Component | Description |
|-----------|-------------|
| **Publisher** | Client/server that sends messages to channels |
| **Subscriber** | Client/server that receives messages from channels |
| **Channel** | Named conduit through which messages are routed |
| **Message** | Data payload (JSON, string) up to 32KB |
| **PubNub Network** | Global broker handling routing and delivery |

### Decoupling Principles

- **Space Decoupling**: Publishers and subscribers don't need to know each other
- **Time Decoupling**: Messages can be sent when no subscribers are online
- **Synchronization Decoupling**: Publishers don't block waiting for receivers

## Publishing Messages

### Basic Publish (JavaScript)

```javascript
const pubnub = new PubNub({
  publishKey: 'pub-c-...',
  subscribeKey: 'sub-c-...',
  userId: 'user-123'
});

// Publish JSON object
await pubnub.publish({
  channel: 'chat-room',
  message: {
    user: 'Alice',
    text: 'Hello, World!',
    timestamp: Date.now()
  }
});
```

### Python SDK

```python
from pubnub.pnconfiguration import PNConfiguration
from pubnub.pubnub import PubNub

pnconfig = PNConfiguration()
pnconfig.publish_key = 'pub-c-...'
pnconfig.subscribe_key = 'sub-c-...'
pnconfig.uuid = 'user-123'

pubnub = PubNub(pnconfig)

pubnub.publish().channel('chat-room').message({
    'user': 'Alice',
    'text': 'Hello, World!'
}).sync()
```

### Publish with Callback (Async)

```javascript
pubnub.publish(
  {
    channel: 'notifications',
    message: { type: 'alert', content: 'New update available' }
  },
  (status, response) => {
    if (!status.error) {
      console.log('Published with timetoken:', response.timetoken);
    } else {
      console.error('Publish error:', status);
    }
  }
);
```

## Subscribing to Messages

### Basic Subscribe with Listener

```javascript
// Add listener BEFORE subscribing
pubnub.addListener({
  message: (event) => {
    console.log('Received:', event.message);
    console.log('Channel:', event.channel);
    console.log('Publisher:', event.publisher);
    console.log('Timetoken:', event.timetoken);
  },
  status: (statusEvent) => {
    if (statusEvent.category === 'PNConnectedCategory') {
      console.log('Connected successfully');
    }
  }
});

// Subscribe to channels
pubnub.subscribe({
  channels: ['chat-room', 'notifications']
});
```

### Python Subscribe

```python
from pubnub.callbacks import SubscribeCallback

class MySubscribeCallback(SubscribeCallback):
    def message(self, pubnub, message):
        print(f'Received: {message.message}')
        print(f'Channel: {message.channel}')

    def status(self, pubnub, status):
        print(f'Status: {status.category}')

pubnub.add_listener(MySubscribeCallback())
pubnub.subscribe().channels(['chat-room']).execute()
```

## Message Delivery Behavior

### When No Subscribers Are Present

- Messages are still accepted by PubNub
- Short-term cache holds last 100 messages (~20 minutes)
- With Message Persistence enabled, messages are stored for configured retention
- Reconnecting clients can catch up using timetokens

### Catching Up on Missed Messages

```javascript
// Subscribe with restore option for automatic catch-up
pubnub.subscribe({
  channels: ['chat-room'],
  withPresence: false
});

// Manual history fetch for longer gaps
const history = await pubnub.fetchMessages({
  channels: ['chat-room'],
  count: 100
});
```

## Best Practices

### Message Payload Design

```javascript
// Good: Small, focused payload
{
  "type": "chat_message",
  "user": "alice",
  "text": "Hello",
  "ts": 1704067200000
}

// Avoid: Large, nested structures
// Keep under 32KB, use concise field names
```

### Error Handling

```javascript
pubnub.addListener({
  status: (statusEvent) => {
    switch (statusEvent.category) {
      case 'PNConnectedCategory':
        console.log('Connected');
        break;
      case 'PNReconnectedCategory':
        console.log('Reconnected');
        break;
      case 'PNDisconnectedCategory':
        console.warn('Disconnected - will retry');
        break;
      case 'PNNetworkDownCategory':
        console.error('Network down');
        break;
      case 'PNAccessDeniedCategory':
        console.error('Access denied - check permissions');
        break;
    }
  }
});
```

### Unsubscribe and Cleanup

```javascript
// Unsubscribe from specific channels
pubnub.unsubscribe({
  channels: ['chat-room']
});

// Unsubscribe from all
pubnub.unsubscribeAll();

// Clean up on page unload
window.addEventListener('beforeunload', () => {
  pubnub.unsubscribeAll();
});
```

## Communication Patterns

| Pattern | Description | Example |
|---------|-------------|---------|
| **Fan-out** | One publisher to many subscribers | Broadcast announcements |
| **Fan-in** | Many publishers to one channel | Sensor data aggregation |
| **1:1 Chat** | Two users, dedicated channel | Direct messaging |
| **Group Chat** | Multiple users, shared channel | Team chat rooms |
| **Request/Response** | Publish request, await reply | RPC-style patterns |

## Performance Guidelines

- Publish rate: 10-15 messages/second per channel recommended
- Message size: Keep well under 32KB for optimal delivery
- Subscriber capacity: Ensure clients can process incoming message rate
- Use async publishing to avoid blocking
