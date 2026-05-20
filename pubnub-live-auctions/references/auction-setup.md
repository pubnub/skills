<!-- xrefs-injected -->

> **Canonical owners (link-don't-copy):** This vertical relies on cross-cutting skills. Always link to the canonical owner instead of duplicating. Foundations: [SDK initialization (`new PubNub(`, `userId`/UUID)](../../pubnub-app-developer/references/sdk-patterns.md), [pub/sub basics (`pubnub.publish(`, `pubnub.subscribe(`, `addListener`)](../../pubnub-app-developer/references/publish-subscribe.md), [channel naming](../../pubnub-app-developer/references/channels.md), [message filters](../../pubnub-app-developer/references/message-filters.md), [SDK upgrades](../../pubnub-app-developer/references/sdk-upgrades.md), [REST API](../../pubnub-app-developer/references/rest-api.md). Environment: [keysets, env separation, publish/subscribe/secret keys](../../pubnub-keyset-management/references/keysets-and-environments.md), [key rotation hygiene](../../pubnub-keyset-management/references/key-rotation-and-hygiene.md), [demo keys](../../pubnub-keyset-management/references/demo-keys.md), [custom origin](../../pubnub-keyset-management/references/custom-origin.md). Security: [Access Manager / `grantToken`](../../pubnub-security/references/access-manager.md), [AES-256 / message encryption](../../pubnub-security/references/encryption.md), [IP allowlisting](../../pubnub-security/references/ip-whitelisting.md), [DoS mitigation](../../pubnub-security/references/dos-mitigation.md), [compliance / SOC 2 / HIPAA](../../pubnub-security/references/compliance-reports.md). Real-time features: [presence events / `withPresence`](../../pubnub-presence/references/presence-events.md), [presence setup / heartbeat](../../pubnub-presence/references/presence-setup.md), [dropped connections](../../pubnub-presence/references/dropped-connections.md), [multi-device sync](../../pubnub-presence/references/multi-device-sync.md). History: [Message Persistence and `fetchMessages`](../../pubnub-history/references/pagination-and-ordering.md), [offline catch-up](../../pubnub-history/references/offline-catch-up.md), [retention](../../pubnub-history/references/retention-and-storage.md). App Context: [users / user metadata](../../pubnub-app-context/references/users.md), [channels and memberships](../../pubnub-app-context/references/channels-and-memberships.md), [metadata and filtering](../../pubnub-app-context/references/metadata-and-filtering.md). Functions: [Before/After Publish, `request.ok()`/`request.abort()`](../../pubnub-functions/references/functions-basics.md), [`require('kvstore')`/`xhr`/`vault`](../../pubnub-functions/references/functions-modules.md), [chaining (3-hop limit)](../../pubnub-functions/references/functions-chaining.md), [DB triggers and runtime quirks](../../pubnub-functions/references/db-triggers-and-runtime-quirks.md), [common patterns](../../pubnub-functions/references/functions-patterns.md). Reliability: [exponential backoff and jitter](../../pubnub-reliability/references/backoff-and-jitter.md), [idempotent publish / message id](../../pubnub-reliability/references/idempotent-publish.md), [dedup on merge](../../pubnub-reliability/references/dedup-on-merge.md), [queue and retry](../../pubnub-reliability/references/queue-and-retry.md), [schema version](../../pubnub-reliability/references/schema-versioning.md). Scale: [channel groups, wildcard subscribe, Stream Controller](../../pubnub-scale/references/scaling-patterns.md), [performance tuning](../../pubnub-scale/references/performance.md), [10K+ live events](../../pubnub-scale/references/large-events.md). Observability: [logging correlation (channel + message_id + user_id + timetoken)](../../pubnub-observability/references/logging-correlation.md), [test pyramid](../../pubnub-observability/references/test-pyramid.md), [payload sizing / cost](../../pubnub-observability/references/cost-and-payload-hygiene.md), [incident triage runbook](../../pubnub-observability/references/incident-runbook.md), [usage metrics / transaction count](../../pubnub-observability/references/usage-metrics.md). Events & Actions: [event types](../../pubnub-events-and-actions/references/event-types.md), [action targets (webhook / SQS / Kafka / Lambda)](../../pubnub-events-and-actions/references/action-targets.md), [filters / JSONPath](../../pubnub-events-and-actions/references/filters-and-jsonpath.md). Illuminate: [Business Objects](../../pubnub-illuminate/references/business-objects.md), [Metrics](../../pubnub-illuminate/references/metrics.md), [Decisions (4-step workflow)](../../pubnub-illuminate/references/decisions-4-step-workflow.md), [Queries](../../pubnub-illuminate/references/queries-adhoc-vs-saved.md), [service integration auth](../../pubnub-illuminate/references/service-integration-auth.md). Chat: [Chat SDK setup](../../pubnub-chat/references/chat-setup.md), [message actions / reactions](../../pubnub-chat/references/message-actions.md), [file sharing / `sendFile`](../../pubnub-chat/references/file-sharing.md), [threading](../../pubnub-chat/references/threading.md). Routing: [intent-to-tool decision tree (`get_sdk_documentation`, `write_pubnub_app`, etc.)](../../pubnub-choose-docs-path/references/intent-to-tool.md).


# PubNub Live Auction Setup

## Overview

This reference covers the foundational architecture for building real-time auction platforms with PubNub, including channel design, auction lifecycle management, timer synchronization, and SDK initialization patterns.

## Installation

### JavaScript/TypeScript

```bash
npm install pubnub
# or
yarn add pubnub
```

### Prerequisites

- Node.js >= 16.0.0
- Publish and Subscribe keys from PubNub Admin Portal
- **PubNub Functions** enabled for server-side bid validation
- **Message Persistence** enabled for bid history and audit trails
- **App Context** enabled if tracking user metadata

## SDK Initialization

### Client-Side (Bidder)

```javascript
import PubNub from 'pubnub';

const pubnub = new PubNub({
  publishKey: 'pub-c-...',
  subscribeKey: 'sub-c-...',
  userId: 'bidder-alice-001',
  authKey: 'bidder-auth-token',   // For Access Manager
  restore: true,                   // Reconnect and catch up on missed messages
  retryConfiguration: PubNub.LinearRetryPolicy({
    delay: 2,
    maximumRetry: 5
  })
});
```

### Server-Side (Auction Admin)

```javascript
import PubNub from 'pubnub';

const pubnub = new PubNub({
  publishKey: 'pub-c-...',
  subscribeKey: 'sub-c-...',
  secretKey: 'sec-c-...',         // Server-only, never expose to clients
  userId: 'auction-server'
});
```

## Channel Design

### Channel Naming Conventions

| Channel Pattern | Purpose | Example |
|----------------|---------|---------|
| `auction.<itemId>` | Live bid updates for a specific auction | `auction.item-5001` |
| `auction.<itemId>.activity` | Bid history feed, notifications | `auction.item-5001.activity` |
| `auction.<itemId>.admin` | Admin controls (start, pause, cancel) | `auction.item-5001.admin` |
| `catalog.active` | Currently active auction listings | `catalog.active` |
| `catalog.upcoming` | Scheduled auctions not yet started | `catalog.upcoming` |
| `catalog.completed` | Recently completed auctions | `catalog.completed` |
| `user.<userId>.notifications` | Personal outbid and win notifications | `user.bidder-alice-001.notifications` |

### Channel Architecture Diagram

```
catalog.active ──────── Broadcasts active auction summaries
catalog.upcoming ────── Broadcasts upcoming auction previews

auction.item-5001 ───── Bid updates (validated bids only)
  ├── .activity ──────── Bid history, join/leave, milestones
  └── .admin ─────────── Start/pause/extend/cancel controls

user.<userId>.notifications ── Personal bid alerts
```

### Subscribing to Auction Channels

```javascript
// Bidder subscribes to relevant channels
function joinAuction(pubnub, auctionId) {
  pubnub.subscribe({
    channels: [
      `auction.${auctionId}`,
      `auction.${auctionId}.activity`
    ],
    withPresence: true   // Track how many bidders are watching
  });
}

// Browse catalog of active auctions
function browseCatalog(pubnub) {
  pubnub.subscribe({
    channels: ['catalog.active', 'catalog.upcoming']
  });
}

// Admin subscribes to control channels
function joinAsAdmin(pubnub, auctionId) {
  pubnub.subscribe({
    channels: [
      `auction.${auctionId}`,
      `auction.${auctionId}.activity`,
      `auction.${auctionId}.admin`
    ]
  });
}
```

## Auction Lifecycle Management

### Auction States

| State | Description | Transitions To |
|-------|-------------|----------------|
| `draft` | Auction created but not published | `scheduled`, `cancelled` |
| `scheduled` | Published with a future start time | `active`, `cancelled` |
| `active` | Open for bidding | `closing`, `paused`, `cancelled` |
| `closing` | Final countdown (last 60 seconds) | `completed`, `active` (if auto-extended) |
| `paused` | Temporarily halted by admin | `active`, `cancelled` |
| `completed` | Bidding ended, winner determined | `archived` |
| `cancelled` | Auction voided | `archived` |
| `archived` | Historical record | (terminal) |

### Creating an Auction

```javascript
async function createAuction(pubnub, auctionData) {
  const auction = {
    id: `item-${Date.now()}`,
    title: auctionData.title,
    description: auctionData.description,
    imageUrls: auctionData.imageUrls || [],
    startingPrice: auctionData.startingPrice,
    reservePrice: auctionData.reservePrice || null,
    minimumIncrement: auctionData.minimumIncrement || 1.00,
    startTime: auctionData.startTime,     // ISO 8601 timestamp
    endTime: auctionData.endTime,         // ISO 8601 timestamp
    state: 'scheduled',
    currentBid: null,
    currentBidderId: null,
    bidCount: 0,
    autoExtendSeconds: auctionData.autoExtendSeconds || 30,
    createdAt: new Date().toISOString()
  };

  // Publish to upcoming catalog
  await pubnub.publish({
    channel: 'catalog.upcoming',
    message: {
      type: 'auction_scheduled',
      auction: auction
    },
    storeInHistory: true
  });

  return auction;
}
```

### Starting an Auction

```javascript
async function startAuction(pubnub, auction) {
  auction.state = 'active';

  // Move from upcoming to active catalog
  await pubnub.publish({
    channel: 'catalog.active',
    message: {
      type: 'auction_started',
      auction: {
        id: auction.id,
        title: auction.title,
        startingPrice: auction.startingPrice,
        endTime: auction.endTime,
        state: 'active'
      }
    },
    storeInHistory: true
  });

  // Notify the auction channel
  await pubnub.publish({
    channel: `auction.${auction.id}`,
    message: {
      type: 'auction_started',
      auctionId: auction.id,
      startingPrice: auction.startingPrice,
      endTime: auction.endTime,
      remainingMs: new Date(auction.endTime).getTime() - Date.now()
    }
  });

  // Start countdown timer
  startCountdownBroadcast(pubnub, auction);
}
```

### Ending an Auction

```javascript
async function endAuction(pubnub, auction) {
  auction.state = 'completed';

  const result = {
    type: 'auction_ended',
    auctionId: auction.id,
    winnerId: auction.currentBidderId,
    winningBid: auction.currentBid,
    bidCount: auction.bidCount,
    reserveMet: auction.reservePrice
      ? auction.currentBid >= auction.reservePrice
      : true
  };

  // Broadcast to auction channel
  await pubnub.publish({
    channel: `auction.${auction.id}`,
    message: result,
    storeInHistory: true
  });

  // Update catalog
  await pubnub.publish({
    channel: 'catalog.active',
    message: {
      type: 'auction_removed',
      auctionId: auction.id
    }
  });

  await pubnub.publish({
    channel: 'catalog.completed',
    message: {
      type: 'auction_completed',
      auction: { ...auction, result }
    },
    storeInHistory: true
  });

  // Notify winner
  if (result.winnerId && result.reserveMet) {
    await pubnub.publish({
      channel: `user.${result.winnerId}.notifications`,
      message: {
        type: 'auction_won',
        auctionId: auction.id,
        title: auction.title,
        amount: result.winningBid
      }
    });
  }
}
```

## Timer Synchronization

### Server-Authoritative Countdown

Client clocks cannot be trusted. The server must be the authoritative source of remaining time for every auction.

```javascript
function startCountdownBroadcast(pubnub, auction) {
  const endTime = new Date(auction.endTime).getTime();

  const intervalId = setInterval(async () => {
    const remainingMs = endTime - Date.now();

    if (remainingMs <= 0) {
      clearInterval(intervalId);
      await endAuction(pubnub, auction);
      return;
    }

    // Broadcast countdown every second during closing phase,
    // every 10 seconds otherwise
    const isClosing = remainingMs <= 60000;
    auction.state = isClosing ? 'closing' : 'active';

    await pubnub.publish({
      channel: `auction.${auction.id}`,
      message: {
        type: 'countdown',
        auctionId: auction.id,
        remainingMs: remainingMs,
        state: auction.state
      }
    });
  }, auction.state === 'closing' ? 1000 : 10000);

  return intervalId;
}
```

### Client-Side Countdown Rendering

```javascript
class AuctionCountdown {
  constructor(pubnub, auctionId, displayElement) {
    this.auctionId = auctionId;
    this.displayElement = displayElement;
    this.serverRemainingMs = null;
    this.lastServerUpdate = null;
    this.localInterval = null;

    pubnub.addListener({
      message: (event) => {
        if (event.channel === `auction.${auctionId}`) {
          this.handleMessage(event.message);
        }
      }
    });
  }

  handleMessage(msg) {
    if (msg.type === 'countdown') {
      this.serverRemainingMs = msg.remainingMs;
      this.lastServerUpdate = Date.now();
      this.startLocalInterpolation();
    }
    if (msg.type === 'auction_ended') {
      this.stop();
      this.displayElement.textContent = 'ENDED';
    }
  }

  startLocalInterpolation() {
    if (this.localInterval) return;

    this.localInterval = setInterval(() => {
      const elapsed = Date.now() - this.lastServerUpdate;
      const estimated = Math.max(0, this.serverRemainingMs - elapsed);

      const minutes = Math.floor(estimated / 60000);
      const seconds = Math.floor((estimated % 60000) / 1000);
      this.displayElement.textContent =
        `${minutes}:${seconds.toString().padStart(2, '0')}`;

      if (estimated <= 0) {
        this.stop();
      }
    }, 250);
  }

  stop() {
    if (this.localInterval) {
      clearInterval(this.localInterval);
      this.localInterval = null;
    }
  }
}
```

## Admin Panel Patterns

### Auction Admin Controls

```javascript
async function pauseAuction(pubnub, auctionId) {
  await pubnub.publish({
    channel: `auction.${auctionId}.admin`,
    message: { type: 'pause', auctionId, adminId: pubnub.getUserId() }
  });

  await pubnub.publish({
    channel: `auction.${auctionId}`,
    message: {
      type: 'auction_paused',
      auctionId,
      reason: 'Administrative hold'
    }
  });
}

async function resumeAuction(pubnub, auctionId, newEndTime) {
  await pubnub.publish({
    channel: `auction.${auctionId}`,
    message: {
      type: 'auction_resumed',
      auctionId,
      endTime: newEndTime,
      remainingMs: new Date(newEndTime).getTime() - Date.now()
    }
  });
}

async function cancelAuction(pubnub, auctionId, reason) {
  await pubnub.publish({
    channel: `auction.${auctionId}`,
    message: {
      type: 'auction_cancelled',
      auctionId,
      reason: reason
    },
    storeInHistory: true
  });

  await pubnub.publish({
    channel: 'catalog.active',
    message: { type: 'auction_removed', auctionId }
  });
}
```

### Monitoring Active Auctions

```javascript
// Use Presence to track active bidders
pubnub.addListener({
  presence: (event) => {
    if (event.action === 'join') {
      console.log(`${event.uuid} joined ${event.channel}`);
      updateBidderCount(event.channel, event.occupancy);
    }
    if (event.action === 'leave' || event.action === 'timeout') {
      updateBidderCount(event.channel, event.occupancy);
    }
  }
});

function updateBidderCount(channel, occupancy) {
  const auctionId = channel.replace('auction.', '');
  document.getElementById(`watchers-${auctionId}`).textContent =
    `${occupancy} watching`;
}
```

## Access Manager Configuration

### Grant Permissions

```javascript
// Server-side: grant bidder access to auction channels
async function grantBidderAccess(pubnub, userId, auctionId) {
  await pubnub.grant({
    channels: [
      `auction.${auctionId}`,
      `auction.${auctionId}.activity`
    ],
    uuids: [userId],
    read: true,
    write: true,     // Allow publishing bids
    ttl: 60          // Minutes
  });

  // Personal notification channel
  await pubnub.grant({
    channels: [`user.${userId}.notifications`],
    uuids: [userId],
    read: true,
    write: false,    // Only server can write notifications
    ttl: 1440        // 24 hours
  });
}

// Grant admin full access
async function grantAdminAccess(pubnub, adminId, auctionId) {
  await pubnub.grant({
    channels: [
      `auction.${auctionId}`,
      `auction.${auctionId}.activity`,
      `auction.${auctionId}.admin`
    ],
    uuids: [adminId],
    read: true,
    write: true,
    manage: true,
    ttl: 1440
  });
}
```

## React Integration

```javascript
import { useState, useEffect } from 'react';
import PubNub from 'pubnub';

function AuctionApp({ userId }) {
  const [pubnub, setPubnub] = useState(null);

  useEffect(() => {
    const pn = new PubNub({
      publishKey: process.env.REACT_APP_PUBNUB_PUB_KEY,
      subscribeKey: process.env.REACT_APP_PUBNUB_SUB_KEY,
      userId: userId,
      restore: true
    });

    setPubnub(pn);

    return () => {
      pn.unsubscribeAll();
      pn.destroy();
    };
  }, [userId]);

  if (!pubnub) return <div>Loading...</div>;

  return <AuctionDashboard pubnub={pubnub} />;
}
```

## Best Practices

1. **Channel naming** - Use dot-separated hierarchical names (`auction.item-5001.activity`) for easy wildcard subscriptions and access control.
2. **Message persistence** - Enable `storeInHistory: true` for all bid and auction state messages to create audit trails.
3. **Presence tracking** - Use PubNub Presence on auction channels to display the number of watchers, which creates urgency.
4. **Graceful reconnection** - Configure `restore: true` and retry policies so bidders automatically reconnect and catch up on missed bids.
5. **Server-authoritative time** - Never let the client determine auction start/end times or remaining countdown values.
6. **Access Manager** - Restrict admin channels to admin users only; grant bidders read+write on auction channels with short TTLs.
7. **Channel cleanup** - Unsubscribe from auction channels when a user leaves an auction page to reduce unnecessary message traffic.
8. **Idempotent processing** - Use message timetokens to deduplicate bids on the client side in case of network retries.
