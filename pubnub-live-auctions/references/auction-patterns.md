# PubNub Auction Patterns

## Overview

This reference covers advanced auction patterns built on PubNub, including reserve prices, auto-extend timers for anti-sniping protection, catalog browsing with real-time updates, multi-item auctions, proxy bidding, push notifications, and analytics.

## Reserve Price Implementation

### Setting a Reserve Price

The reserve price is the minimum amount the seller will accept. It is stored server-side and never revealed to bidders.

```javascript
// Server-side: store auction with reserve price
async function createAuctionWithReserve(pubnub, kvstore, auctionData) {
  const auctionState = {
    id: auctionData.id,
    title: auctionData.title,
    startingPrice: auctionData.startingPrice,
    reservePrice: auctionData.reservePrice,   // Hidden from clients
    minimumIncrement: auctionData.minimumIncrement,
    currentBid: null,
    currentBidderId: null,
    bidCount: 0,
    reserveMet: false,
    state: 'active',
    endTime: auctionData.endTime
  };

  await kvstore.set(`auction:${auctionData.id}`, auctionState);
  return auctionState;
}
```

### Reserve Check in PubNub Function

```javascript
// Inside Before Publish handler after accepting a bid
function checkReserve(auctionState, newBidAmount) {
  const wasMet = auctionState.reserveMet;
  auctionState.reserveMet = auctionState.reservePrice
    ? newBidAmount >= auctionState.reservePrice
    : true;

  // Return whether reserve status changed
  return !wasMet && auctionState.reserveMet;
}

// After bid accepted, if reserve status changed:
if (reserveJustMet) {
  pubnub.publish({
    channel: `auction.${auctionId}`,
    message: {
      type: 'reserve_met',
      auctionId: auctionId,
      message: 'Reserve price has been met!'
    }
  });
}
```

### Client-Side Reserve Display

```javascript
function ReserveIndicator({ auctionId, pubnub }) {
  const [reserveMet, setReserveMet] = useState(false);

  useEffect(() => {
    const listener = {
      message: (event) => {
        if (event.channel === `auction.${auctionId}`) {
          if (event.message.type === 'reserve_met') {
            setReserveMet(true);
          }
        }
      }
    };
    pubnub.addListener(listener);
    return () => pubnub.removeListener(listener);
  }, [pubnub, auctionId]);

  return (
    <div className={`reserve-status ${reserveMet ? 'met' : 'not-met'}`}>
      {reserveMet ? 'Reserve Met' : 'Reserve Not Met'}
    </div>
  );
}
```

## Auto-Extend Timers (Anti-Sniping Protection)

### The Problem

Sniping is when a bidder places a bid in the final seconds, leaving no time for others to respond. Auto-extend adds time whenever a bid arrives in the final moments.

### Server-Side Auto-Extend Logic

```javascript
// PubNub Function: check if bid triggers auto-extend
function checkAutoExtend(auctionState, bidTime) {
  const endTime = new Date(auctionState.endTime).getTime();
  const remainingMs = endTime - bidTime;
  const extendThresholdMs = (auctionState.autoExtendSeconds || 30) * 1000;

  if (remainingMs <= extendThresholdMs && remainingMs > 0) {
    // Extend the auction
    const newEndTime = bidTime + extendThresholdMs;
    auctionState.endTime = new Date(newEndTime).toISOString();
    auctionState.extensionCount = (auctionState.extensionCount || 0) + 1;

    return {
      extended: true,
      newEndTime: auctionState.endTime,
      remainingMs: extendThresholdMs,
      extensionNumber: auctionState.extensionCount
    };
  }

  return { extended: false };
}

// Inside bid acceptance flow
const extensionResult = checkAutoExtend(auctionState, Date.now());

if (extensionResult.extended) {
  // Broadcast the extension
  pubnub.publish({
    channel: `auction.${auctionId}`,
    message: {
      type: 'auction_extended',
      auctionId: auctionId,
      newEndTime: extensionResult.newEndTime,
      remainingMs: extensionResult.remainingMs,
      extensionNumber: extensionResult.extensionNumber,
      reason: 'Bid received during final countdown'
    }
  });
}
```

### Client-Side Extension Handling

```javascript
pubnub.addListener({
  message: (event) => {
    const msg = event.message;

    if (msg.type === 'auction_extended') {
      // Update the countdown timer
      updateCountdownTarget(msg.auctionId, msg.newEndTime, msg.remainingMs);

      // Show extension notification
      showToast(`Auction extended! ${msg.remainingMs / 1000}s added.`);
    }
  }
});
```

### Extension Limits

| Setting | Recommended Value | Purpose |
|---------|------------------|---------|
| `autoExtendSeconds` | 30-60 | Time added per extension |
| `extendThresholdSeconds` | 30-120 | Window before end that triggers extension |
| `maxExtensions` | 10-50 | Maximum number of extensions allowed |
| `maxTotalDurationHours` | 24-48 | Hard stop regardless of extensions |

```javascript
// Enforce extension limits in PubNub Function
function checkAutoExtendWithLimits(auctionState, bidTime) {
  const maxExtensions = auctionState.maxExtensions || 20;
  const currentExtensions = auctionState.extensionCount || 0;

  if (currentExtensions >= maxExtensions) {
    return { extended: false, reason: 'Maximum extensions reached' };
  }

  // Check hard time limit
  const createdAt = new Date(auctionState.createdAt).getTime();
  const maxDurationMs = (auctionState.maxTotalDurationHours || 24) * 3600000;
  if (bidTime - createdAt > maxDurationMs) {
    return { extended: false, reason: 'Maximum auction duration reached' };
  }

  return checkAutoExtend(auctionState, bidTime);
}
```

## Catalog Browsing with Real-Time Updates

### Publishing Catalog Updates

```javascript
// Server publishes catalog snapshots periodically
async function publishCatalogUpdate(pubnub, activeAuctions) {
  const summaries = activeAuctions.map(auction => ({
    id: auction.id,
    title: auction.title,
    imageUrl: auction.imageUrls[0],
    currentBid: auction.currentBid || auction.startingPrice,
    bidCount: auction.bidCount,
    endTime: auction.endTime,
    reserveMet: auction.reserveMet,
    watcherCount: auction.watcherCount
  }));

  await pubnub.publish({
    channel: 'catalog.active',
    message: {
      type: 'catalog_snapshot',
      auctions: summaries,
      timestamp: Date.now()
    }
  });
}
```

### Client-Side Catalog with Live Price Updates

```javascript
function AuctionCatalog({ pubnub }) {
  const [auctions, setAuctions] = useState([]);

  useEffect(() => {
    pubnub.subscribe({ channels: ['catalog.active'] });

    const listener = {
      message: (event) => {
        const msg = event.message;

        if (msg.type === 'catalog_snapshot') {
          setAuctions(msg.auctions);
        }

        if (msg.type === 'catalog_price_update') {
          setAuctions(prev => prev.map(a =>
            a.id === msg.auctionId
              ? { ...a, currentBid: msg.currentBid, bidCount: msg.bidCount }
              : a
          ));
        }

        if (msg.type === 'auction_removed') {
          setAuctions(prev => prev.filter(a => a.id !== msg.auctionId));
        }
      }
    };

    pubnub.addListener(listener);

    return () => {
      pubnub.removeListener(listener);
      pubnub.unsubscribe({ channels: ['catalog.active'] });
    };
  }, [pubnub]);

  return (
    <div className="catalog-grid">
      {auctions.map(auction => (
        <AuctionCard key={auction.id} auction={auction} />
      ))}
    </div>
  );
}
```

## Auction Type Comparison

| Feature | English (Ascending) | Dutch (Descending) | Sealed Bid | Buy Now |
|---------|--------------------|--------------------|------------|---------|
| Starting price | Low | High | N/A | Fixed |
| Price direction | Increases | Decreases | Hidden | Static |
| Winner | Highest bidder | First bidder | Highest sealed | First buyer |
| Real-time updates | Bid stream | Price ticks | None until reveal | Inventory count |
| PubNub channel pattern | `auction.<id>` | `dutch.<id>` | `sealed.<id>` | `buynow.<id>` |
| Timer behavior | Count down to end | Price drops on ticks | Count down to reveal | No timer |

## Multi-Item Auctions

### Lot-Based Auctions

```javascript
// Create a multi-lot auction
async function createMultiLotAuction(pubnub, lots) {
  const auctionGroupId = `group-${Date.now()}`;

  for (const lot of lots) {
    await pubnub.publish({
      channel: 'catalog.active',
      message: {
        type: 'auction_scheduled',
        auction: {
          id: lot.id,
          groupId: auctionGroupId,
          lotNumber: lot.lotNumber,
          title: lot.title,
          startingPrice: lot.startingPrice,
          startTime: lot.startTime,
          state: 'scheduled'
        }
      },
      storeInHistory: true
    });
  }

  // Subscribe to group channel for sequential lot management
  pubnub.subscribe({
    channels: [`group.${auctionGroupId}`]
  });
}

// Transition to next lot when current lot ends
async function advanceToNextLot(pubnub, groupId, currentLotNumber, lots) {
  const nextLot = lots.find(l => l.lotNumber === currentLotNumber + 1);
  if (!nextLot) {
    await pubnub.publish({
      channel: `group.${groupId}`,
      message: { type: 'all_lots_completed', groupId }
    });
    return;
  }

  await pubnub.publish({
    channel: `group.${groupId}`,
    message: {
      type: 'next_lot',
      lotNumber: nextLot.lotNumber,
      auctionId: nextLot.id,
      title: nextLot.title,
      startingPrice: nextLot.startingPrice
    }
  });
}
```

## Proxy Bidding (Auto-Bidding)

### How Proxy Bidding Works

A bidder sets a maximum amount. The system automatically places the minimum necessary bid on their behalf whenever they are outbid, up to their maximum.

### Server-Side Proxy Bid Logic

```javascript
// PubNub Function: process proxy bids after a new bid is accepted
function processProxyBids(kvstore, pubnub, auctionId, newBidAmount) {
  const proxyKey = `proxy:${auctionId}`;

  return kvstore.get(proxyKey).then((proxyBids) => {
    if (!proxyBids || !proxyBids.bids || proxyBids.bids.length === 0) {
      return Promise.resolve();
    }

    // Find proxy bids that can counter
    const eligibleProxies = proxyBids.bids
      .filter(p => p.maxAmount > newBidAmount && p.active)
      .sort((a, b) => b.maxAmount - a.maxAmount);

    if (eligibleProxies.length === 0) return Promise.resolve();

    const topProxy = eligibleProxies[0];

    return kvstore.get(`auction:${auctionId}`).then((auctionState) => {
      const minimumIncrement = auctionState.minimumIncrement;
      const autoBidAmount = Math.min(
        newBidAmount + minimumIncrement,
        topProxy.maxAmount
      );

      // Place the proxy bid
      return pubnub.publish({
        channel: `auction.${auctionId}`,
        message: {
          type: 'bid',
          bidderId: topProxy.bidderId,
          auctionId: auctionId,
          amount: autoBidAmount,
          isProxyBid: true,
          timestamp: Date.now()
        }
      });
    });
  });
}
```

### Client-Side Proxy Bid Setup

```javascript
async function setProxyBid(pubnub, auctionId, maxAmount) {
  await pubnub.publish({
    channel: `auction.${auctionId}.admin`,
    message: {
      type: 'set_proxy_bid',
      bidderId: pubnub.getUserId(),
      auctionId: auctionId,
      maxAmount: maxAmount,
      timestamp: Date.now()
    }
  });
}

// UI component
function ProxyBidForm({ pubnub, auctionId, currentBid, minimumIncrement }) {
  const [maxBid, setMaxBid] = useState('');

  const handleSubmit = async (e) => {
    e.preventDefault();
    const amount = parseFloat(maxBid);

    if (amount <= currentBid + minimumIncrement) {
      alert(`Maximum bid must be greater than $${(currentBid + minimumIncrement).toFixed(2)}`);
      return;
    }

    await setProxyBid(pubnub, auctionId, amount);
    alert('Proxy bid set. The system will bid on your behalf.');
  };

  return (
    <form onSubmit={handleSubmit}>
      <label>Set maximum bid:</label>
      <input
        type="number"
        step="0.01"
        value={maxBid}
        onChange={(e) => setMaxBid(e.target.value)}
        placeholder={`Min: $${(currentBid + minimumIncrement).toFixed(2)}`}
      />
      <button type="submit">Set Auto-Bid</button>
    </form>
  );
}
```

## Push Notification Patterns

### Mobile Push for Bid Alerts

```javascript
// Server-side: register device for push notifications
async function registerForAuctionPush(pubnub, userId, deviceToken, platform) {
  const channels = [`user.${userId}.notifications`];

  if (platform === 'apns') {
    await pubnub.push.addChannels({
      channels: channels,
      device: deviceToken,
      pushGateway: 'apns2',
      environment: 'production',
      topic: 'com.yourapp.auctions'
    });
  } else if (platform === 'fcm') {
    await pubnub.push.addChannels({
      channels: channels,
      device: deviceToken,
      pushGateway: 'gcm'
    });
  }
}
```

### iOS Push Payload (Swift)

```swift
// PubNub Function: format push notification for outbid
// Attach push payload when publishing outbid notification
{
  "pn_apns": {
    "aps": {
      "alert": {
        "title": "You've been outbid!",
        "body": "Vintage Watch: New bid $350. Bid $360+ to retake the lead."
      },
      "badge": 1,
      "sound": "bid_alert.caf",
      "category": "OUTBID_ACTION"
    },
    "auctionId": "item-5001",
    "bidAmount": 350
  },
  "pn_gcm": {
    "notification": {
      "title": "You've been outbid!",
      "body": "Vintage Watch: New bid $350. Bid $360+ to retake the lead."
    },
    "data": {
      "auctionId": "item-5001",
      "bidAmount": "350",
      "action": "OUTBID"
    }
  }
}
```

### Android Notification Handler (Kotlin)

```kotlin
class AuctionNotificationService : FirebaseMessagingService() {
    override fun onMessageReceived(remoteMessage: RemoteMessage) {
        val data = remoteMessage.data
        val auctionId = data["auctionId"] ?: return
        val action = data["action"] ?: return

        when (action) {
            "OUTBID" -> showOutbidNotification(auctionId, data)
            "AUCTION_WON" -> showWinNotification(auctionId, data)
            "ENDING_SOON" -> showEndingSoonNotification(auctionId, data)
        }
    }

    private fun showOutbidNotification(auctionId: String, data: Map<String, String>) {
        val bidAmount = data["bidAmount"]?.toDoubleOrNull() ?: return

        val intent = Intent(this, AuctionActivity::class.java).apply {
            putExtra("auctionId", auctionId)
            flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TASK
        }

        val notification = NotificationCompat.Builder(this, "auction_alerts")
            .setContentTitle("You've been outbid!")
            .setContentText("New bid: $$bidAmount. Tap to bid again.")
            .setSmallIcon(R.drawable.ic_bid)
            .setContentIntent(PendingIntent.getActivity(this, 0, intent, PendingIntent.FLAG_IMMUTABLE))
            .setAutoCancel(true)
            .build()

        NotificationManagerCompat.from(this).notify(auctionId.hashCode(), notification)
    }
}
```

## Auction Analytics and Reporting

### Tracking Auction Metrics

```javascript
// PubNub Function: After Publish handler for analytics
export default (request) => {
  const kvstore = require('kvstore');
  const message = request.message;

  if (message.type === 'bid_accepted') {
    const analyticsKey = `analytics:${message.auctionId}`;

    return kvstore.get(analyticsKey).then((stats) => {
      if (!stats) {
        stats = {
          totalBids: 0,
          uniqueBidders: [],
          bidAmounts: [],
          firstBidTime: null,
          lastBidTime: null,
          peakBiddingRate: 0
        };
      }

      stats.totalBids += 1;
      stats.lastBidTime = Date.now();
      stats.firstBidTime = stats.firstBidTime || Date.now();

      if (!stats.uniqueBidders.includes(message.bidderId)) {
        stats.uniqueBidders.push(message.bidderId);
      }

      stats.bidAmounts.push(message.amount);

      return kvstore.set(analyticsKey, stats);
    }).then(() => request.ok());
  }

  return request.ok();
};
```

### Analytics Dashboard Data

```javascript
async function getAuctionAnalytics(kvstore, auctionId) {
  const stats = await kvstore.get(`analytics:${auctionId}`);
  if (!stats) return null;

  const amounts = stats.bidAmounts;
  const durationMs = stats.lastBidTime - stats.firstBidTime;

  return {
    totalBids: stats.totalBids,
    uniqueBidders: stats.uniqueBidders.length,
    averageBidAmount: amounts.reduce((a, b) => a + b, 0) / amounts.length,
    highestBid: Math.max(...amounts),
    lowestBid: Math.min(...amounts),
    biddingDurationMinutes: Math.round(durationMs / 60000),
    bidsPerMinute: durationMs > 0
      ? (stats.totalBids / (durationMs / 60000)).toFixed(2)
      : 0,
    priceIncrease: amounts.length > 1
      ? ((amounts[amounts.length - 1] - amounts[0]) / amounts[0] * 100).toFixed(1)
      : 0
  };
}
```

### Analytics Event Table

| Event | Channel | Key Data | Purpose |
|-------|---------|----------|---------|
| `bid_accepted` | `auction.<id>` | amount, bidderId, bidNumber | Track bidding velocity |
| `auction_started` | `catalog.active` | startingPrice, endTime | Track auction inventory |
| `auction_extended` | `auction.<id>` | extensionNumber | Measure sniping frequency |
| `reserve_met` | `auction.<id>` | bidAmount | Conversion tracking |
| `auction_ended` | `auction.<id>` | winningBid, bidCount | Revenue and engagement |
| `presence_join` | `auction.<id>-pnpres` | occupancy | Track watcher interest |

## Dutch Auction Pattern

In a Dutch auction, the price starts high and drops at regular intervals until someone bids.

```javascript
// Server-side: run Dutch auction price drops
function startDutchAuction(pubnub, auction) {
  const dropIntervalMs = auction.dropIntervalSeconds * 1000;
  let currentPrice = auction.startingPrice;
  const floorPrice = auction.floorPrice;
  const dropAmount = auction.dropAmount;

  const intervalId = setInterval(async () => {
    currentPrice -= dropAmount;

    if (currentPrice <= floorPrice) {
      currentPrice = floorPrice;
      clearInterval(intervalId);
    }

    await pubnub.publish({
      channel: `dutch.${auction.id}`,
      message: {
        type: 'price_drop',
        auctionId: auction.id,
        currentPrice: currentPrice,
        floorPrice: floorPrice,
        nextDropIn: dropIntervalMs
      }
    });

    if (currentPrice <= floorPrice) {
      await pubnub.publish({
        channel: `dutch.${auction.id}`,
        message: {
          type: 'floor_reached',
          auctionId: auction.id,
          finalPrice: floorPrice
        }
      });
    }
  }, dropIntervalMs);

  return intervalId;
}

// Client bids at current price to win immediately
async function bidDutchAuction(pubnub, auctionId) {
  await pubnub.publish({
    channel: `dutch.${auctionId}`,
    message: {
      type: 'dutch_bid',
      bidderId: pubnub.getUserId(),
      auctionId: auctionId,
      timestamp: Date.now()
    }
  });
}
```

## Watchlist Pattern

```javascript
// Client-side: maintain a watchlist of auctions
async function addToWatchlist(pubnub, auctionId) {
  const userId = pubnub.getUserId();
  const watchlistChannel = `user.${userId}.notifications`;

  // Subscribe to ending-soon notifications for this auction
  pubnub.subscribe({
    channels: [`auction.${auctionId}`, watchlistChannel]
  });

  // Store in local watchlist
  const watchlist = JSON.parse(localStorage.getItem('watchlist') || '[]');
  if (!watchlist.includes(auctionId)) {
    watchlist.push(auctionId);
    localStorage.setItem('watchlist', JSON.stringify(watchlist));
  }
}

// Server-side: notify watchers when auction is ending
async function notifyWatchers(pubnub, auctionId, remainingMinutes) {
  // Fetch watcher list from your database
  const watchers = await getAuctionWatchers(auctionId);

  for (const userId of watchers) {
    await pubnub.publish({
      channel: `user.${userId}.notifications`,
      message: {
        type: 'auction_ending_soon',
        auctionId: auctionId,
        auctionTitle: auction.title,
        remainingMinutes: remainingMinutes,
        currentBid: auction.currentBid
      }
    });
  }
}
```

## Best Practices

1. **Reserve prices** - Store reserve prices exclusively in KV Store server-side. Only broadcast whether the reserve has been met, never the actual reserve amount.
2. **Anti-sniping** - Use auto-extend timers with reasonable limits (e.g., 30-second extensions, max 20 extensions) to ensure fair bidding.
3. **Proxy bidding security** - Process proxy bids server-side in PubNub Functions. Never send a bidder's maximum amount to other clients.
4. **Catalog efficiency** - Publish catalog snapshots periodically rather than on every bid. Use separate price-update messages for live pricing.
5. **Push notification throttling** - Consolidate rapid outbid notifications to avoid overwhelming users. Send at most one push notification per auction per minute.
6. **Multi-item sequencing** - For lot-based auctions, use a group channel to coordinate lot transitions so all clients advance together.
7. **Analytics isolation** - Use After Publish handlers for analytics to avoid blocking or slowing down bid processing in Before Publish handlers.
8. **Dutch auction atomicity** - Validate Dutch auction bids server-side to ensure only the first bidder wins when multiple bid at the same price tick.
9. **Watchlist scalability** - For large platforms, use channel groups instead of individual subscriptions to manage user watchlists efficiently.
10. **Extension transparency** - Always broadcast extension events so all bidders know the timer has been reset, maintaining trust in the auction process.
