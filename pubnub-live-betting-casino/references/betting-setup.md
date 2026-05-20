<!-- xrefs-injected -->

> **Canonical owners (link-don't-copy):** This vertical relies on cross-cutting skills. Always link to the canonical owner instead of duplicating. Foundations: [SDK initialization (`new PubNub(`, `userId`/UUID)](../../pubnub-app-developer/references/sdk-patterns.md), [pub/sub basics (`pubnub.publish(`, `pubnub.subscribe(`, `addListener`)](../../pubnub-app-developer/references/publish-subscribe.md), [channel naming](../../pubnub-app-developer/references/channels.md), [message filters](../../pubnub-app-developer/references/message-filters.md), [SDK upgrades](../../pubnub-app-developer/references/sdk-upgrades.md), [REST API](../../pubnub-app-developer/references/rest-api.md). Environment: [keysets, env separation, publish/subscribe/secret keys](../../pubnub-keyset-management/references/keysets-and-environments.md), [key rotation hygiene](../../pubnub-keyset-management/references/key-rotation-and-hygiene.md), [demo keys](../../pubnub-keyset-management/references/demo-keys.md), [custom origin](../../pubnub-keyset-management/references/custom-origin.md). Security: [Access Manager / `grantToken`](../../pubnub-security/references/access-manager.md), [AES-256 / message encryption](../../pubnub-security/references/encryption.md), [IP allowlisting](../../pubnub-security/references/ip-whitelisting.md), [DoS mitigation](../../pubnub-security/references/dos-mitigation.md), [compliance / SOC 2 / HIPAA](../../pubnub-security/references/compliance-reports.md). Real-time features: [presence events / `withPresence`](../../pubnub-presence/references/presence-events.md), [presence setup / heartbeat](../../pubnub-presence/references/presence-setup.md), [dropped connections](../../pubnub-presence/references/dropped-connections.md), [multi-device sync](../../pubnub-presence/references/multi-device-sync.md). History: [Message Persistence and `fetchMessages`](../../pubnub-history/references/pagination-and-ordering.md), [offline catch-up](../../pubnub-history/references/offline-catch-up.md), [retention](../../pubnub-history/references/retention-and-storage.md). App Context: [users / user metadata](../../pubnub-app-context/references/users.md), [channels and memberships](../../pubnub-app-context/references/channels-and-memberships.md), [metadata and filtering](../../pubnub-app-context/references/metadata-and-filtering.md). Functions: [Before/After Publish, `request.ok()`/`request.abort()`](../../pubnub-functions/references/functions-basics.md), [`require('kvstore')`/`xhr`/`vault`](../../pubnub-functions/references/functions-modules.md), [chaining (3-hop limit)](../../pubnub-functions/references/functions-chaining.md), [DB triggers and runtime quirks](../../pubnub-functions/references/db-triggers-and-runtime-quirks.md), [common patterns](../../pubnub-functions/references/functions-patterns.md). Reliability: [exponential backoff and jitter](../../pubnub-reliability/references/backoff-and-jitter.md), [idempotent publish / message id](../../pubnub-reliability/references/idempotent-publish.md), [dedup on merge](../../pubnub-reliability/references/dedup-on-merge.md), [queue and retry](../../pubnub-reliability/references/queue-and-retry.md), [schema version](../../pubnub-reliability/references/schema-versioning.md). Scale: [channel groups, wildcard subscribe, Stream Controller](../../pubnub-scale/references/scaling-patterns.md), [performance tuning](../../pubnub-scale/references/performance.md), [10K+ live events](../../pubnub-scale/references/large-events.md). Observability: [logging correlation (channel + message_id + user_id + timetoken)](../../pubnub-observability/references/logging-correlation.md), [test pyramid](../../pubnub-observability/references/test-pyramid.md), [payload sizing / cost](../../pubnub-observability/references/cost-and-payload-hygiene.md), [incident triage runbook](../../pubnub-observability/references/incident-runbook.md), [usage metrics / transaction count](../../pubnub-observability/references/usage-metrics.md). Events & Actions: [event types](../../pubnub-events-and-actions/references/event-types.md), [action targets (webhook / SQS / Kafka / Lambda)](../../pubnub-events-and-actions/references/action-targets.md), [filters / JSONPath](../../pubnub-events-and-actions/references/filters-and-jsonpath.md). Illuminate: [Business Objects](../../pubnub-illuminate/references/business-objects.md), [Metrics](../../pubnub-illuminate/references/metrics.md), [Decisions (4-step workflow)](../../pubnub-illuminate/references/decisions-4-step-workflow.md), [Queries](../../pubnub-illuminate/references/queries-adhoc-vs-saved.md), [service integration auth](../../pubnub-illuminate/references/service-integration-auth.md). Chat: [Chat SDK setup](../../pubnub-chat/references/chat-setup.md), [message actions / reactions](../../pubnub-chat/references/message-actions.md), [file sharing / `sendFile`](../../pubnub-chat/references/file-sharing.md), [threading](../../pubnub-chat/references/threading.md). Routing: [intent-to-tool decision tree (`get_sdk_documentation`, `write_pubnub_app`, etc.)](../../pubnub-choose-docs-path/references/intent-to-tool.md).


# PubNub Betting Platform Setup

## Overview

This reference covers initializing a PubNub-powered betting platform, designing market channel hierarchies, broadcasting odds in real time, and securing the infrastructure with Access Manager and encryption.

## SDK Installation

### JavaScript/TypeScript

```bash
npm install pubnub
# or
yarn add pubnub
```

### Prerequisites

- Publish and Subscribe keys from PubNub Admin Portal
- **Access Manager** enabled on the keyset
- **Message Persistence** enabled for bet audit trails
- **PubNub Functions** enabled for server-side validation

## Basic Initialization

```javascript
import PubNub from 'pubnub';

const pubnub = new PubNub({
  publishKey: 'pub-c-...',
  subscribeKey: 'sub-c-...',
  userId: 'betting-client-001',
  cipherKey: 'your-encryption-key',
  authKey: 'auth-token-from-server'
});
```

## Configuration Options

```javascript
const pubnub = new PubNub({
  publishKey: 'pub-c-...',
  subscribeKey: 'sub-c-...',
  userId: 'betting-client-001',

  // Security
  cipherKey: 'aes-256-encryption-key',   // Encrypt all messages
  authKey: 'pam-auth-token',             // Access Manager token

  // Connection
  ssl: true,                              // Always use TLS
  keepAlive: true,                        // Persistent connections
  presenceTimeout: 120,                   // Detect disconnected users

  // Performance
  requestMessageCountThreshold: 100,      // Message queue threshold
  restore: true,                          // Reconnect and catch up
  autoNetworkDetection: true              // Detect network changes
});
```

## Market Channel Design

### Channel Naming Convention

Betting platforms require a structured channel hierarchy to organize events, markets, and selections efficiently.

| Channel Pattern | Purpose | Example |
|----------------|---------|---------|
| `event.{sport}.{eventId}` | Event-level updates (scores, status) | `event.football.12345` |
| `event.{sport}.{eventId}.market.{marketId}` | Market-level odds updates | `event.football.12345.market.match-winner` |
| `odds.{sport}.live` | Aggregated live odds feed per sport | `odds.football.live` |
| `wagers.submit` | Wager submission channel | `wagers.submit` |
| `wagers.{userId}.status` | Per-user bet status updates | `wagers.user-789.status` |
| `balance.{userId}` | Per-user balance updates | `balance.user-789` |

### Channel Groups

Use channel groups to manage subscriptions efficiently when users follow multiple events.

```javascript
// Server-side: Add market channels to an event channel group
await pubnub.channelGroups.addChannels({
  channelGroup: 'event-football-12345-markets',
  channels: [
    'event.football.12345.market.match-winner',
    'event.football.12345.market.over-under-2-5',
    'event.football.12345.market.both-teams-score',
    'event.football.12345.market.correct-score'
  ]
});

// Client-side: Subscribe to all markets for an event
pubnub.subscribe({
  channelGroups: ['event-football-12345-markets']
});
```

## Odds Broadcasting

### Odds Format Types

| Format | Example | Description |
|--------|---------|-------------|
| Decimal | 2.50 | Multiply by stake for total return |
| Fractional | 3/2 | Profit-to-stake ratio (UK standard) |
| American | +150 | Positive = profit on $100, Negative = stake for $100 profit |

### Publishing Odds Updates

```javascript
async function publishOdds(eventId, market) {
  await pubnub.publish({
    channel: `event.football.${eventId}.market.${market.id}`,
    message: {
      type: 'odds_update',
      marketId: market.id,
      marketName: market.name,
      selections: market.selections.map(sel => ({
        id: sel.id,
        name: sel.name,
        odds: {
          decimal: sel.decimal,
          fractional: sel.fractional,
          american: sel.american
        },
        movement: sel.movement,       // 'up', 'down', or 'stable'
        status: sel.status             // 'active', 'suspended', 'resulted'
      })),
      suspended: market.suspended,
      inPlay: market.inPlay,
      timestamp: Date.now(),
      sequence: market.sequenceNumber  // For ordering guarantees
    }
  });
}
```

### Odds Conversion Utility

```javascript
function decimalToFractional(decimal) {
  const numerator = Math.round((decimal - 1) * 100);
  const denominator = 100;
  const gcd = (a, b) => (b ? gcd(b, a % b) : a);
  const d = gcd(numerator, denominator);
  return `${numerator / d}/${denominator / d}`;
}

function decimalToAmerican(decimal) {
  if (decimal >= 2.0) {
    return `+${Math.round((decimal - 1) * 100)}`;
  } else {
    return `${Math.round(-100 / (decimal - 1))}`;
  }
}
```

### Market Suspension

When key events happen (goals, red cards, injuries), markets must be suspended before new odds are calculated.

```javascript
async function suspendMarket(eventId, marketId, reason) {
  await pubnub.publish({
    channel: `event.football.${eventId}.market.${marketId}`,
    message: {
      type: 'market_suspension',
      marketId: marketId,
      suspended: true,
      reason: reason,           // 'goal', 'red_card', 'var_review', 'manual'
      timestamp: Date.now()
    }
  });
}

// Resume market with new odds
async function resumeMarket(eventId, market) {
  market.suspended = false;
  await publishOdds(eventId, market);
}
```

## Platform Security

### Access Manager Configuration

```javascript
// Server-side: Grant permissions
// Odds engine gets publish access to market channels
await pubnub.grant({
  channels: ['event.football.*.market.*'],
  authKeys: ['odds-engine-auth'],
  read: false,
  write: true,
  ttl: 60  // Minutes
});

// Users get read-only access to market channels
await pubnub.grant({
  channels: ['event.football.*.market.*'],
  authKeys: ['user-auth-token'],
  read: true,
  write: false,
  ttl: 1440  // 24 hours
});

// Users get write access to wager submission only
await pubnub.grant({
  channels: ['wagers.submit'],
  authKeys: ['user-auth-token'],
  read: false,
  write: true,
  ttl: 1440
});
```

### Token-Based Authentication

```javascript
async function generateUserToken(userId) {
  const token = await pubnub.grantToken({
    ttl: 1440,
    authorized_uuid: userId,
    resources: {
      channels: {
        [`wagers.${userId}.status`]: { read: true },
        [`balance.${userId}`]: { read: true },
        'wagers.submit': { write: true }
      },
      channelGroups: {
        'live-football': { read: true },
        'live-basketball': { read: true }
      }
    }
  });
  return token;
}

// Client receives token and sets it
const pubnub = new PubNub({
  subscribeKey: 'sub-c-...',
  userId: 'user-789'
});
pubnub.setToken(authToken);
```

## Encryption

```javascript
// All messages on betting channels are encrypted by default
const pubnub = new PubNub({
  publishKey: 'pub-c-...',
  subscribeKey: 'sub-c-...',
  userId: 'betting-client-001',
  cipherKey: 'your-256-bit-encryption-key'
});
```

### Custom Encryption for Sensitive Data

```javascript
import crypto from 'crypto';

function encryptSensitive(data, key) {
  const iv = crypto.randomBytes(16);
  const cipher = crypto.createCipheriv('aes-256-cbc', key, iv);
  let encrypted = cipher.update(JSON.stringify(data), 'utf8', 'hex');
  encrypted += cipher.final('hex');
  return { encrypted, iv: iv.toString('hex') };
}
```

## Swift Initialization (iOS)

```swift
import PubNub

let config = PubNubConfiguration(
  publishKey: "pub-c-...",
  subscribeKey: "sub-c-...",
  userId: "ios-user-123"
)
config.cipherKey = Crypto(key: "encryption-key")
config.authKey = "auth-token-from-server"

let pubnub = PubNub(configuration: config)

pubnub.subscribe(to: ["event.football.12345.market.match-winner"])

let listener = SubscriptionListener()
listener.didReceiveMessage = { message in
  guard let oddsUpdate = message.payload.dictionaryOptional else { return }
  DispatchQueue.main.async {
    self.updateOddsDisplay(oddsUpdate)
  }
}
pubnub.add(listener)
```

## Kotlin Initialization (Android)

```kotlin
import com.pubnub.api.PubNub
import com.pubnub.api.PNConfiguration
import com.pubnub.api.UserId

val config = PNConfiguration(userId = UserId("android-user-456")).apply {
    publishKey = "pub-c-..."
    subscribeKey = "sub-c-..."
    cipherKey = "encryption-key"
    authKey = "auth-token-from-server"
    secure = true
}

val pubnub = PubNub.create(config)

pubnub.subscribe(channels = listOf("event.football.12345.market.match-winner"))

pubnub.addListener(object : SubscribeCallback() {
    override fun message(pubnub: PubNub, pnMessageResult: PNMessageResult) {
        val odds = pnMessageResult.message
        runOnUiThread { updateOddsDisplay(odds) }
    }
})
```

## Regulatory Considerations

- **Audit Trail**: Enable Message Persistence to store all odds changes and wager messages for regulatory review
- **Geo-Fencing**: Verify user location before granting access to betting channels (see `betting-patterns.md`)
- **Age Verification**: Require identity verification before issuing Access Manager tokens
- **Data Retention**: Configure message retention policies to meet jurisdictional requirements (e.g., 5 years in UK)
- **Responsible Gambling**: Integrate with self-exclusion registries before granting channel access

## Error Handling

```javascript
pubnub.addListener({
  status: (statusEvent) => {
    switch (statusEvent.category) {
      case 'PNConnectedCategory':
        console.log('Connected to betting channels');
        break;
      case 'PNAccessDeniedCategory':
        console.error('Access denied - refresh auth token');
        refreshAuthToken();
        break;
      case 'PNNetworkIssuesCategory':
        console.warn('Network issue - odds may be stale');
        showStaleOddsWarning();
        break;
      case 'PNReconnectedCategory':
        console.log('Reconnected - refreshing odds');
        refreshAllOdds();
        break;
    }
  }
});
```

## Best Practices

1. **Use channel groups** to manage market subscriptions instead of subscribing to channels individually
2. **Include sequence numbers** in odds messages to handle out-of-order delivery
3. **Always encrypt** wager and balance channels with AES-256 encryption
4. **Enable Message Persistence** on all channels for regulatory audit trails
5. **Set short TTLs** on Access Manager tokens and rotate frequently
6. **Use TLS/SSL** for all connections without exception
7. **Implement reconnection logic** that refreshes stale odds on network recovery
8. **Separate read and write permissions** so clients cannot publish to odds channels
9. **Use presence** to track active users for responsible gambling monitoring
10. **Design channel names** with consistent dot-separated hierarchies for easy pattern matching
