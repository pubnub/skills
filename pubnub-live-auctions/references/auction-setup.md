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
    message: { type: 'pause', auctionId, adminId: pubnub.getUUID() }
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
