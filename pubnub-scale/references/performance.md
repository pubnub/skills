<!-- canonical-for: PERFORMANCE_TUNING -->
<!-- used-by: -->

> **Cross-references:** Cost-side [payload sizing, fan-out hygiene, coalescing, and signal vs publish](../../pubnub-observability/references/cost-and-payload-hygiene.md) is owned by `pubnub-observability` — link there for cost-driven decisions. Built on [pub/sub basics (`pubnub.publish(`, `pubnub.subscribe(`)](../../pubnub-app-developer/references/publish-subscribe.md) and [SDK initialization (`new PubNub(`, `userId`/UUID)](../../pubnub-app-developer/references/sdk-patterns.md). For [Message Persistence](../../pubnub-history/references/pagination-and-ordering.md) see `pubnub-history`. For [reconnect with backoff under load](../../pubnub-reliability/references/backoff-and-jitter.md) see `pubnub-reliability`. For Function-side optimizations see [Before Publish patterns](../../pubnub-functions/references/functions-basics.md).

# PubNub Performance Optimization

## Message Optimization

### Size Limits

| Limit | Value |
|-------|-------|
| Maximum message size | 32KB (32,768 bytes) |
| Recommended size | < 2KB for optimal speed |
| Channel name | 92 characters max |

### Calculating Message Size

```javascript
// Estimate message size
function estimateMessageSize(channel, message) {
  const json = JSON.stringify(message);
  const encoded = encodeURIComponent(channel + json);
  return encoded.length + 100;  // Add overhead
}

// Check before publish
const size = estimateMessageSize('my-channel', myMessage);
if (size > 32000) {
  console.warn('Message too large:', size);
}
```

### Payload Optimization

```javascript
// BAD: Verbose payload (wasteful)
{
  "messageType": "user_activity_update",
  "userIdentifier": "alice-smith-123",
  "activityStatus": "online",
  "lastActiveTimestamp": 1704067200000,
  "additionalMetadata": {
    "deviceInformation": "iPhone 14 Pro",
    "applicationVersion": "2.1.0"
  }
}

// GOOD: Compact payload (efficient)
{
  "t": "activity",
  "u": "alice-123",
  "s": "on",
  "ts": 1704067200000,
  "m": {
    "d": "iPhone14Pro",
    "v": "2.1.0"
  }
}
```

### Compression Strategies

```javascript
// 1. Use short field names
const fieldMap = {
  type: 't',
  userId: 'u',
  status: 's',
  timestamp: 'ts'
};

// 2. Use numeric codes instead of strings
const statusCodes = {
  online: 1,
  offline: 0,
  away: 2,
  busy: 3
};

// 3. Remove null/undefined fields
function compactMessage(message) {
  return Object.fromEntries(
    Object.entries(message).filter(([_, v]) => v != null)
  );
}

// 4. Consider delta updates (send only changes)
function createDelta(previous, current) {
  const delta = {};
  for (const [key, value] of Object.entries(current)) {
    if (previous[key] !== value) {
      delta[key] = value;
    }
  }
  return delta;
}
```

## Publish Rate Optimization

### Recommended Rates

| Scenario | Messages/Second/Channel |
|----------|------------------------|
| Optimal | 10-15 |
| Maximum sustainable | 30-50 |
| Burst (short periods) | 100+ |

### Batching Messages

```javascript
// Instead of many small publishes
async function publishBatch(channel, items) {
  // Batch items into single message
  await pubnub.publish({
    channel,
    message: {
      type: 'batch',
      items,
      timestamp: Date.now()
    }
  });
}

// Usage
const updates = [
  { id: 1, value: 10 },
  { id: 2, value: 20 },
  { id: 3, value: 30 }
];
await publishBatch('sensor-updates', updates);
```

### Async Publishing

```javascript
// Use async publish to avoid blocking
const publishPromises = messages.map(msg =>
  pubnub.publish({ channel: 'updates', message: msg })
);

// Wait for all to complete
const results = await Promise.all(publishPromises);

// Or fire-and-forget for non-critical
messages.forEach(msg => {
  pubnub.publish({ channel: 'analytics', message: msg })
    .catch(err => console.error('Publish failed:', err));
});
```

## Subscribe Optimization

### Message Queue

Each channel has a short-term buffer:
- **Default**: 100 messages
- **Duration**: ~20 minutes
- **Behavior**: FIFO (old messages pushed out)

```javascript
// If subscriber can't keep up, messages may be lost
// Solutions:

// 1. Batch messages on publish side
// 2. Use Message Persistence for catch-up
// 3. Contact PubNub to increase queue size
```

### Efficient Listeners

```javascript
// BAD: Heavy processing in listener
pubnub.addListener({
  message: (event) => {
    // Don't do heavy work here
    processHeavyData(event.message);  // Blocks next message
    updateDatabase(event.message);
    sendNotifications(event.message);
  }
});

// GOOD: Queue and process asynchronously
const messageQueue = [];

pubnub.addListener({
  message: (event) => {
    messageQueue.push(event);
  }
});

// Process queue in background
setInterval(() => {
  const batch = messageQueue.splice(0, 100);
  if (batch.length > 0) {
    processBatch(batch);  // Non-blocking
  }
}, 100);
```

### Filtering Messages

```javascript
// Client-side filtering for specific message types
pubnub.addListener({
  message: (event) => {
    const msg = event.message;

    // Early exit for irrelevant messages
    if (msg.type !== 'update') return;
    if (msg.userId === currentUserId) return;  // Skip own messages

    // Process relevant messages
    handleUpdate(msg);
  }
});
```

## Network Optimization

### HTTP Pipelining

PubNub supports HTTP pipelining - multiple requests on same connection without waiting for responses.

```javascript
// SDKs handle this automatically
// Multiple publishes can be pipelined
await Promise.all([
  pubnub.publish({ channel: 'ch1', message: msg1 }),
  pubnub.publish({ channel: 'ch2', message: msg2 }),
  pubnub.publish({ channel: 'ch3', message: msg3 })
]);
```

### Connection Reuse

```javascript
// Create single PubNub instance per client
// DON'T create new instances per operation

// BAD
async function sendMessage(message) {
  const pn = new PubNub({ /* config */ });  // New instance each time
  await pn.publish({ channel: 'chat', message });
}

// GOOD
const pubnub = new PubNub({ /* config */ });  // Single instance

async function sendMessage(message) {
  await pubnub.publish({ channel: 'chat', message });
}
```

## Mobile Optimization

### Battery-Conscious Configuration

```javascript
const pubnub = new PubNub({
  subscribeKey: 'sub-c-...',
  userId: 'user-123',
  // Reduce heartbeat frequency for battery savings
  heartbeatInterval: 120,    // 2 minutes instead of default
  presenceTimeout: 600,      // 10 minutes timeout
  // Disable features not needed
  suppressLeaveEvents: true  // Reduce presence traffic
});
```

### Background Mode

```javascript
// Reduce activity when app is backgrounded
document.addEventListener('visibilitychange', () => {
  if (document.hidden) {
    // App in background
    pubnub.unsubscribe({
      channels: nonCriticalChannels
    });
  } else {
    // App in foreground
    pubnub.subscribe({
      channels: nonCriticalChannels
    });
  }
});
```

## Performance Monitoring

### Track Publish Latency

```javascript
async function timedPublish(channel, message) {
  const start = performance.now();

  const result = await pubnub.publish({ channel, message });

  const latency = performance.now() - start;
  console.log(`Publish latency: ${latency.toFixed(2)}ms`);

  return result;
}
```

### Monitor Message Rate

```javascript
class MessageRateMonitor {
  constructor() {
    this.counts = new Map();
    this.interval = setInterval(() => this.report(), 60000);
  }

  record(channel) {
    const count = this.counts.get(channel) || 0;
    this.counts.set(channel, count + 1);
  }

  report() {
    for (const [channel, count] of this.counts) {
      console.log(`${channel}: ${count} msgs/min`);
    }
    this.counts.clear();
  }
}

const monitor = new MessageRateMonitor();

pubnub.addListener({
  message: (event) => {
    monitor.record(event.channel);
  }
});
```
