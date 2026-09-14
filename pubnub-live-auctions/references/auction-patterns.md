# PubNub Auction Patterns

Reserve, auto-extend, proxy (KV + Function), catalog channels, watchlists, and push payloads. Auction-type economics primers (English/Dutch/sealed how-it-works), React UIs, and install manuals are out of scope.

## Reserve price (Function)

Store `reservePrice` in KV only. Broadcast `reserve_met`, never the reserve amount.

```javascript
async function createAuctionWithReserve(kvstore, auctionData) {
  const auctionState = {
    id: auctionData.id,
    startingPrice: auctionData.startingPrice,
    reservePrice: auctionData.reservePrice,
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

function checkReserve(auctionState, newBidAmount) {
  const wasMet = auctionState.reserveMet;
  auctionState.reserveMet = auctionState.reservePrice
    ? newBidAmount >= auctionState.reservePrice
    : true;
  return !wasMet && auctionState.reserveMet;
}
```

After accept, if reserve just met:

```javascript
pubnub.publish({
  channel: `auction.${auctionId}`,
  message: { type: 'reserve_met', auctionId, message: 'Reserve price has been met!' }
});
```

## Auto-extend (server publish)

When a bid arrives in the final window, extend `endTime` in KV and publish `auction_extended`.

```javascript
function checkAutoExtend(auctionState, bidTime) {
  const endTime = new Date(auctionState.endTime).getTime();
  const remainingMs = endTime - bidTime;
  const extendThresholdMs = (auctionState.autoExtendSeconds || 30) * 1000;

  if (remainingMs <= extendThresholdMs && remainingMs > 0) {
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
```

```javascript
if (extensionResult.extended) {
  pubnub.publish({
    channel: `auction.${auctionId}`,
    message: {
      type: 'auction_extended',
      auctionId,
      newEndTime: extensionResult.newEndTime,
      remainingMs: extensionResult.remainingMs,
      extensionNumber: extensionResult.extensionNumber
    }
  });
}
```

| Setting | Typical bound | Purpose |
|---------|----------------|---------|
| `autoExtendSeconds` | 30-60 | Time added per extension |
| `maxExtensions` | operator cap | Stop infinite sniping loops |
| `maxTotalDurationHours` | operator cap | Hard stop |

Clients update countdown from `auction_extended` + `countdown` ticks ([auction-setup.md](auction-setup.md)).

## Catalog channels

```javascript
async function publishCatalogUpdate(pubnub, activeAuctions) {
  const summaries = activeAuctions.map(auction => ({
    id: auction.id,
    title: auction.title,
    currentBid: auction.currentBid || auction.startingPrice,
    bidCount: auction.bidCount,
    endTime: auction.endTime,
    reserveMet: auction.reserveMet
  }));

  await pubnub.publish({
    channel: 'catalog.active',
    message: { type: 'catalog_snapshot', auctions: summaries, timestamp: Date.now() }
  });
}
```

Also publish `catalog_price_update` and `auction_removed` on `catalog.active`. Snapshot periodically; do not republish the full catalog on every bid.

### Multi-lot group channel

```javascript
await pubnub.publish({
  channel: `group.${groupId}`,
  message: { type: 'next_lot', lotNumber: nextLot.lotNumber, auctionId: nextLot.id }
});
```

## Proxy bidding (KV + Function)

Store max amounts in KV (`proxy:{auctionId}`). After a new high bid, publish the minimum necessary auto-bid on `auction.{id}` with `isProxyBid: true`. Never send another bidder's max to clients.

```javascript
async function processProxyBids(kvstore, pubnub, auctionId, newBidAmount) {
  const proxyBids = await kvstore.get(`proxy:${auctionId}`);
  if (!proxyBids?.bids?.length) return;
  const eligible = proxyBids.bids.filter(p => p.maxAmount > newBidAmount && p.active)
    .sort((a, b) => b.maxAmount - a.maxAmount);
  if (!eligible.length) return;
  const auctionState = await kvstore.get(`auction:${auctionId}`);
  const autoBidAmount = Math.min(newBidAmount + auctionState.minimumIncrement, eligible[0].maxAmount);
  return pubnub.publish({
    channel: `auction.${auctionId}`,
    message: {
      type: 'bid',
      bidderId: eligible[0].bidderId,
      auctionId,
      amount: autoBidAmount,
      isProxyBid: true,
      timestamp: Date.now()
    }
  });
}
```

```javascript
async function setProxyBid(pubnub, auctionId, maxAmount) {
  await pubnub.publish({
    channel: `auction.${auctionId}.admin`,
    message: {
      type: 'set_proxy_bid',
      bidderId: pubnub.getUserId(),
      auctionId,
      maxAmount,
      timestamp: Date.now()
    }
  });
}
```

Dutch / sealed / buy-now economics are out of scope. If the operator uses a descending-price format, reuse the same Before-Publish + KV lock as [auction-bidding.md](auction-bidding.md) on `dutch.{id}` (first valid bid wins).

## Push payloads (PubNub)

Register `user.{userId}.notifications` with `pubnub.push.addChannels` (`apns2` / `gcm`). Attach `pn_apns` / `pn_gcm` on outbid publishes — retrieve payload shape from MCP / docs. Throttle to at most one push per auction per minute.

## Analytics (After Publish)

After Publish on bid channels: increment KV `analytics:{auctionId}` (`totalBids`, unique bidders). Do not block Before-Publish. Event mapping:

| Event | Channel | Purpose |
|-------|---------|---------|
| `bid_accepted` | `auction.<id>` | Bidding velocity |
| `auction_started` | `catalog.active` | Inventory |
| `auction_extended` | `auction.<id>` | Extension count |
| `reserve_met` | `auction.<id>` | Reserve conversion |
| `auction_ended` | `auction.<id>` | Close-out |
| `presence_join` | `auction.<id>-pnpres` | Watcher occupancy |

## Watchlist

```javascript
async function addToWatchlist(pubnub, auctionId) {
  const userId = pubnub.getUserId();
  pubnub.subscribe({
    channels: [`auction.${auctionId}`, `user.${userId}.notifications`]
  });
}

async function notifyWatchers(pubnub, auctionId, remainingMinutes, auction) {
  const watchers = await getAuctionWatchers(auctionId);
  for (const userId of watchers) {
    await pubnub.publish({
      channel: `user.${userId}.notifications`,
      message: {
        type: 'auction_ending_soon',
        auctionId,
        remainingMinutes,
        currentBid: auction.currentBid
      }
    });
  }
}
```

For large watchlists, use channel groups instead of per-auction subscribe.

## Best Practices

1. **Reserve in KV only**; broadcast met/not-met.
2. **Auto-extend with caps**; always publish `auction_extended`.
3. **Proxy max stays server-side.**
4. **Catalog snapshots + separate price-update messages.**
5. **Throttle push** on rapid outbid storms.
6. **Group channel** for lot sequencing.
7. **Analytics in After Publish**, not Before Publish.
