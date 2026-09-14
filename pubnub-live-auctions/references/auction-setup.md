# PubNub Live Auction Setup

Channel design, lifecycle publish, server countdown ticks, and Access Manager. SDK install manuals and React app shells are out of scope — pull init from MCP.

Enable **Functions** (bid validation), **Message Persistence** (bid audit), and optionally **App Context**. Bidders: `setToken` + `restore: true`. Server: secret key, never on clients.

## Channel design

| Channel Pattern | Purpose | Example |
|----------------|---------|---------|
| `auction.<itemId>` | Live bid updates for a specific auction | `auction.item-5001` |
| `auction.<itemId>.activity` | Bid history feed, notifications | `auction.item-5001.activity` |
| `auction.<itemId>.admin` | Admin controls (start, pause, cancel) | `auction.item-5001.admin` |
| `catalog.active` | Currently active auction listings | `catalog.active` |
| `catalog.upcoming` | Scheduled auctions not yet started | `catalog.upcoming` |
| `catalog.completed` | Recently completed auctions | `catalog.completed` |
| `user.<userId>.notifications` | Personal outbid and win notifications | `user.bidder-alice-001.notifications` |

```
catalog.active / catalog.upcoming
auction.item-5001  (validated bids)
  ├── .activity
  └── .admin
user.<userId>.notifications
```

```javascript
function joinAuction(pubnub, auctionId) {
  pubnub.subscribe({
    channels: [`auction.${auctionId}`, `auction.${auctionId}.activity`],
    withPresence: true
  });
}

function browseCatalog(pubnub) {
  pubnub.subscribe({ channels: ['catalog.active', 'catalog.upcoming'] });
}

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

## Auction lifecycle

| State | Description | Transitions To |
|-------|-------------|----------------|
| `draft` | Created but not published | `scheduled`, `cancelled` |
| `scheduled` | Future start time | `active`, `cancelled` |
| `active` | Open for bidding | `closing`, `paused`, `cancelled` |
| `closing` | Final countdown | `completed`, `active` (if auto-extended) |
| `paused` | Admin hold | `active`, `cancelled` |
| `completed` | Winner determined | `archived` |
| `cancelled` | Voided | `archived` |

### Create / start / end (publish)

```javascript
async function createAuction(pubnub, auctionData) {
  const auction = {
    id: `item-${Date.now()}`,
    startingPrice: auctionData.startingPrice,
    reservePrice: auctionData.reservePrice || null,
    minimumIncrement: auctionData.minimumIncrement || 1.00,
    startTime: auctionData.startTime,
    endTime: auctionData.endTime,
    state: 'scheduled',
    currentBid: null,
    currentBidderId: null,
    bidCount: 0,
    autoExtendSeconds: auctionData.autoExtendSeconds || 30
  };

  await pubnub.publish({
    channel: 'catalog.upcoming',
    message: { type: 'auction_scheduled', auction },
    storeInHistory: true
  });
  return auction;
}

async function startAuction(pubnub, auction) {
  auction.state = 'active';
  await pubnub.publish({
    channel: 'catalog.active',
    message: {
      type: 'auction_started',
      auction: { id: auction.id, startingPrice: auction.startingPrice, endTime: auction.endTime, state: 'active' }
    },
    storeInHistory: true
  });
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
  startCountdownBroadcast(pubnub, auction);
}

async function endAuction(pubnub, auction) {
  auction.state = 'completed';
  const result = {
    type: 'auction_ended',
    auctionId: auction.id,
    winnerId: auction.currentBidderId,
    winningBid: auction.currentBid,
    bidCount: auction.bidCount
  };

  await pubnub.publish({ channel: `auction.${auction.id}`, message: result, storeInHistory: true });
  await pubnub.publish({ channel: 'catalog.active', message: { type: 'auction_removed', auctionId: auction.id } });
  await pubnub.publish({
    channel: 'catalog.completed',
    message: { type: 'auction_completed', auction: { ...auction, result } },
    storeInHistory: true
  });

  if (result.winnerId) {
    await pubnub.publish({
      channel: `user.${result.winnerId}.notifications`,
      message: { type: 'auction_won', auctionId: auction.id, amount: result.winningBid }
    });
  }
}
```

## Server-authoritative countdown

Do not trust client clocks. Publish `countdown` ticks from the server.

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
    const isClosing = remainingMs <= 60000;
    auction.state = isClosing ? 'closing' : 'active';
    await pubnub.publish({
      channel: `auction.${auction.id}`,
      message: {
        type: 'countdown',
        auctionId: auction.id,
        remainingMs,
        state: auction.state
      }
    });
  }, auction.state === 'closing' ? 1000 : 10000);

  return intervalId;
}
```

Clients interpolate locally between ticks; on `auction_extended` ([auction-patterns.md](auction-patterns.md)) reset the target `endTime`.

## Admin publish

Pause / resume / cancel on `auction.{id}.admin` and mirror state on `auction.{id}` / `catalog.active`. Presence occupancy on the auction channel is watcher count.

## Access Manager

```javascript
async function grantBidderAccess(pubnub, userId, auctionId) {
  return pubnub.grantToken({
    ttl: 60,
    authorized_uuid: userId,
    resources: {
      channels: {
        [`auction.${auctionId}`]: { read: true, write: true },
        [`auction.${auctionId}.activity`]: { read: true, write: true },
        [`user.${userId}.notifications`]: { read: true }
      }
    }
  });
}

async function grantAdminAccess(pubnub, adminId, auctionId) {
  return pubnub.grantToken({
    ttl: 1440,
    authorized_uuid: adminId,
    resources: {
      channels: {
        [`auction.${auctionId}`]: { read: true, write: true, manage: true },
        [`auction.${auctionId}.activity`]: { read: true, write: true, manage: true },
        [`auction.${auctionId}.admin`]: { read: true, write: true, manage: true }
      }
    }
  });
}
```

## Best Practices

1. **Dot-separated hierarchy** for wildcards and AM patterns.
2. **`storeInHistory: true`** on bids and state changes.
3. **Presence** for watcher occupancy.
4. **`restore: true`** so bidders catch up after disconnect.
5. **Server time** for start/end/countdown.
6. **Admin channels** restricted; bidder TTLs short.
7. **Unsubscribe** when leaving an auction page.
