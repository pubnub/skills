# PubNub Auction Bidding System

Bid message shapes, Before-Publish validation, CAS/KV delta, outbid channels, and `fetchMessages` history. React bid-history UI and generic race-condition essays are out of scope.

## Bid message shapes

Client submit:

```javascript
{
  type: 'bid',
  bidderId: 'bidder-alice-001',
  auctionId: 'item-5001',
  amount: 150.00,
  timestamp: 1700000000000,
  idempotencyKey: 'bid-abc123-1700000000'
}
```

After Function accept:

```javascript
{
  type: 'bid_accepted',
  bidderId: 'bidder-alice-001',
  auctionId: 'item-5001',
  amount: 150.00,
  previousBid: 140.00,
  previousBidderId: 'bidder-bob-002',
  bidNumber: 17,
  validatedAt: 1700000000123,
  serverTimetoken: '17000000001230000'
}
```

| State | Description | Published To |
|-------|-------------|-------------|
| `bid_submitted` | Client send, awaiting validation | (internal) |
| `bid_accepted` | Now high bid | `auction.<id>` |
| `bid_rejected` | Failed validation | `user.<bidderId>.notifications` |
| `bid_outbid` | Replaced by a higher bid | `user.<bidderId>.notifications` |
| `bid_winning` | Won the auction | `user.<bidderId>.notifications` |

## Before-Publish validation (S1)

Use [server-authoritative Before-Publish](../../pubnub-functions/references/functions-patterns.md). **Auction-specific behavior:**

| Check | Rule |
|-------|------|
| Message type | Only validate `type === 'bid'` |
| Auction state | Reject if not `active` or `closing` |
| Minimum bid | `amount >= currentBid + minimumIncrement` |
| High bidder | Reject if bidder already holds high bid |
| On accept | Update auction KV, transform to `bid_accepted`, notify previous bidder on `user.<id>.notifications` |

### CAS / KV lock (auction delta)

| Step | Auction-specific behavior |
|------|---------------------------|
| Lock | `lock:<auctionId>` with short TTL while processing |
| Read | Current bid + minimum increment from `auction:<auctionId>` KV |
| Reject | Below minimum → `bid_rejected` with updated `currentBid` |
| Accept | Update KV, release lock, emit `bid_accepted` |
| Notify | Publish outbid to `user.<previousBidderId>.notifications` |

Platform structure: [Pattern 1](../../pubnub-functions/references/functions-patterns.md#pattern-1-distributed-counter). Deployable handler: start from that owner, apply the table above. Keep lock TTLs to a few seconds.

## Idempotent bid

Store `idem:<idempotencyKey>` in KV with the prior result; on duplicate, return stored result without re-processing. Same Pattern 1 + auction table.

## Place bid (client)

```javascript
async function placeBid(pubnub, auctionId, amount) {
  const idempotencyKey = `bid-${pubnub.getUserId()}-${Date.now()}`;
  return pubnub.publish({
    channel: `auction.${auctionId}`,
    message: {
      type: 'bid',
      bidderId: pubnub.getUserId(),
      auctionId,
      amount,
      idempotencyKey,
      timestamp: Date.now()
    }
  });
}
```

Listen on `auction.{id}` and `user.{userId}.notifications` for `bid_accepted` / `bid_rejected`. Retry only when the Function signals a retryable lock contention.

Minimum increment is operator-configured (including optional price tiers) and enforced in the Function — not as a product primer here.

## Outbid notifications

```javascript
function sendOutbidNotification(pubnub, previousBidderId, auctionId, details) {
  if (!previousBidderId) return Promise.resolve();
  return pubnub.publish({
    channel: `user.${previousBidderId}.notifications`,
    message: {
      type: 'bid_outbid',
      auctionId,
      yourBid: details.previousBid,
      newHighBid: details.newBid,
      minimumToRegain: details.newBid + details.minimumIncrement,
      timestamp: Date.now()
    }
  });
}
```

```javascript
pubnub.subscribe({ channels: [`user.${pubnub.getUserId()}.notifications`] });
```

Handle `bid_outbid`, `auction_won`, `auction_ending_soon` on that channel.

## Bid history (`fetchMessages`)

```javascript
async function getBidHistory(pubnub, auctionId, count = 100) {
  const response = await pubnub.fetchMessages({
    channels: [`auction.${auctionId}`],
    count,
    includeMessageActions: true
  });

  const messages = response.channels[`auction.${auctionId}`] || [];
  return messages
    .filter(msg => msg.message.type === 'bid_accepted')
    .map(msg => ({
      bidderId: msg.message.bidderId,
      amount: msg.message.amount,
      bidNumber: msg.message.bidNumber,
      timetoken: msg.timetoken
    }));
}
```

Activity feed: publish masked bidder display names to `auction.{id}.activity` with `storeInHistory: true`.

## Bid errors

| Error | Cause | Client action |
|-------|-------|---------------|
| `Bid must be at least X` | Amount below minimum | Show updated minimum |
| `Auction is completed` | Auction ended | Disable bid |
| `Auction is paused` | Admin pause | Wait for resume |
| `You are already the high bidder` | Already leading | No re-bid |
| `Bid processing in progress` | Lock contention | Short retry |
| `Auction not found` | Invalid id | Leave auction |

## Best Practices

1. **Server-side validation only** as authority (S1).
2. **Idempotency keys** per bid attempt.
3. **Short KV locks** for concurrent bids.
4. **Mask bidder ids** on public activity feeds.
5. **`storeInHistory: true`** on accepted bids.
6. **Outbid notify in the same Function transaction** as accept.
