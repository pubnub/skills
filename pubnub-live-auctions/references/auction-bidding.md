# PubNub Auction Bidding System

## Overview

This reference covers the core bidding mechanics for real-time auctions built on PubNub, including server-side bid validation with PubNub Functions, race condition handling, atomic operations with KV Store, outbid notifications, and bid history management.

## Bid Message Structure

### Client-Submitted Bid

```javascript
// Message published by the bidding client
{
  type: 'bid',
  bidderId: 'bidder-alice-001',
  auctionId: 'item-5001',
  amount: 150.00,
  timestamp: 1700000000000,
  idempotencyKey: 'bid-abc123-1700000000'   // Prevents duplicate processing
}
```

### Validated Bid (After PubNub Function)

```javascript
// Message after server-side validation enriches it
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

## Bid States

| State | Description | Published To |
|-------|-------------|-------------|
| `bid_submitted` | Client sends bid, awaiting validation | (internal) |
| `bid_accepted` | Bid validated and is now the current high bid | `auction.<id>` |
| `bid_rejected` | Bid failed validation (too low, auction ended, etc.) | `user.<bidderId>.notifications` |
| `bid_outbid` | A higher bid replaced this one | `user.<bidderId>.notifications` |
| `bid_winning` | This bid won the auction | `user.<bidderId>.notifications` |

## Server-Side Bid Validation with PubNub Functions

### Before Publish Event Handler

PubNub Functions execute server-side on every publish, allowing you to validate bids before they reach other subscribers.

```javascript
// PubNub Function: Before Publish on auction.* channels
export default (request) => {
  const kvstore = require('kvstore');
  const pubnub = require('pubnub');
  const message = request.message;

  // Only validate bid messages
  if (message.type !== 'bid') {
    return request.ok();
  }

  const auctionId = message.auctionId;
  const auctionKey = `auction:${auctionId}`;

  return kvstore.get(auctionKey).then((auctionState) => {
    if (!auctionState) {
      request.message = {
        type: 'bid_rejected',
        reason: 'Auction not found',
        bidderId: message.bidderId,
        auctionId: auctionId
      };
      return request.ok();
    }

    // Check auction is still active
    if (auctionState.state !== 'active' && auctionState.state !== 'closing') {
      request.message = {
        type: 'bid_rejected',
        reason: `Auction is ${auctionState.state}`,
        bidderId: message.bidderId,
        auctionId: auctionId
      };
      return request.ok();
    }

    // Check bid amount
    const currentBid = auctionState.currentBid || auctionState.startingPrice;
    const minimumBid = currentBid + auctionState.minimumIncrement;

    if (message.amount < minimumBid) {
      request.message = {
        type: 'bid_rejected',
        reason: `Bid must be at least ${minimumBid}`,
        currentBid: currentBid,
        minimumBid: minimumBid,
        bidderId: message.bidderId,
        auctionId: auctionId
      };
      return request.ok();
    }

    // Check bidder is not already the high bidder
    if (auctionState.currentBidderId === message.bidderId) {
      request.message = {
        type: 'bid_rejected',
        reason: 'You are already the high bidder',
        bidderId: message.bidderId,
        auctionId: auctionId
      };
      return request.ok();
    }

    // Bid is valid - update auction state
    const previousBidderId = auctionState.currentBidderId;
    const previousBid = auctionState.currentBid;

    auctionState.currentBid = message.amount;
    auctionState.currentBidderId = message.bidderId;
    auctionState.bidCount = (auctionState.bidCount || 0) + 1;
    auctionState.lastBidTime = Date.now();

    return kvstore.set(auctionKey, auctionState).then(() => {
      // Transform message into validated bid
      request.message = {
        type: 'bid_accepted',
        bidderId: message.bidderId,
        auctionId: auctionId,
        amount: message.amount,
        previousBid: previousBid,
        previousBidderId: previousBidderId,
        bidNumber: auctionState.bidCount,
        validatedAt: Date.now()
      };

      // Send outbid notification to previous bidder
      if (previousBidderId) {
        pubnub.publish({
          channel: `user.${previousBidderId}.notifications`,
          message: {
            type: 'bid_outbid',
            auctionId: auctionId,
            yourBid: previousBid,
            newBid: message.amount,
            newBidderId: message.bidderId
          }
        });
      }

      return request.ok();
    });
  });
};
```

## Race Condition Handling

### The Problem

When two bidders submit bids at nearly the same time, both may read the same current bid value from KV Store and both believe their bid is valid.

### Solution: Atomic Compare-and-Set

```javascript
// PubNub Function with compare-and-set pattern
export default (request) => {
  const kvstore = require('kvstore');
  const message = request.message;

  if (message.type !== 'bid') {
    return request.ok();
  }

  const auctionKey = `auction:${message.auctionId}`;
  const lockKey = `lock:${message.auctionId}`;

  // Acquire a simple lock using KV Store
  return kvstore.get(lockKey).then((lock) => {
    if (lock && lock.lockedUntil > Date.now()) {
      // Another bid is being processed - reject with retry hint
      request.message = {
        type: 'bid_rejected',
        reason: 'Bid processing in progress, please retry',
        retryable: true,
        bidderId: message.bidderId,
        auctionId: message.auctionId
      };
      return request.ok();
    }

    // Set lock with 3-second TTL
    return kvstore.set(lockKey, {
      lockedUntil: Date.now() + 3000,
      bidderId: message.bidderId
    }).then(() => {
      return kvstore.get(auctionKey);
    }).then((auctionState) => {
      const currentBid = auctionState.currentBid || auctionState.startingPrice;
      const minimumBid = currentBid + auctionState.minimumIncrement;

      if (message.amount < minimumBid) {
        request.message = {
          type: 'bid_rejected',
          reason: `Bid must be at least ${minimumBid}. Current bid updated while you were bidding.`,
          currentBid: currentBid,
          minimumBid: minimumBid,
          retryable: true,
          bidderId: message.bidderId,
          auctionId: message.auctionId
        };
        // Release lock
        return kvstore.removeItem(lockKey).then(() => request.ok());
      }

      // Update state atomically
      auctionState.currentBid = message.amount;
      auctionState.currentBidderId = message.bidderId;
      auctionState.bidCount = (auctionState.bidCount || 0) + 1;
      auctionState.lastBidTime = Date.now();

      return kvstore.set(auctionKey, auctionState).then(() => {
        return kvstore.removeItem(lockKey);
      }).then(() => {
        request.message = {
          type: 'bid_accepted',
          bidderId: message.bidderId,
          auctionId: message.auctionId,
          amount: message.amount,
          bidNumber: auctionState.bidCount,
          validatedAt: Date.now()
        };
        return request.ok();
      });
    });
  });
};
```

### Client-Side Retry Logic

```javascript
async function placeBidWithRetry(pubnub, auctionId, amount, maxRetries = 3) {
  const idempotencyKey = `bid-${pubnub.getUserId()}-${Date.now()}`;

  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      const result = await pubnub.publish({
        channel: `auction.${auctionId}`,
        message: {
          type: 'bid',
          bidderId: pubnub.getUserId(),
          auctionId: auctionId,
          amount: amount,
          idempotencyKey: idempotencyKey,
          timestamp: Date.now()
        }
      });

      // Wait briefly for the validated response
      return new Promise((resolve, reject) => {
        const timeout = setTimeout(() => {
          reject(new Error('Bid response timeout'));
        }, 5000);

        const listener = {
          message: (event) => {
            const msg = event.message;
            if (msg.auctionId === auctionId && msg.bidderId === pubnub.getUserId()) {
              clearTimeout(timeout);
              pubnub.removeListener(listener);
              if (msg.type === 'bid_accepted') {
                resolve(msg);
              } else if (msg.type === 'bid_rejected') {
                reject(new BidRejectedError(msg.reason, msg));
              }
            }
          }
        };
        pubnub.addListener(listener);
      });

    } catch (error) {
      if (error.retryable && attempt < maxRetries - 1) {
        await new Promise(r => setTimeout(r, 500 * (attempt + 1)));
        continue;
      }
      throw error;
    }
  }
}

class BidRejectedError extends Error {
  constructor(message, details) {
    super(message);
    this.name = 'BidRejectedError';
    this.details = details;
  }
}
```

## Idempotent Bid Processing

Duplicate messages can arrive due to network retries. The PubNub Function must detect and ignore duplicate bids.

```javascript
// Inside PubNub Function: check idempotency key
export default (request) => {
  const kvstore = require('kvstore');
  const message = request.message;

  if (message.type !== 'bid') {
    return request.ok();
  }

  const idempotencyKey = `idem:${message.idempotencyKey}`;

  return kvstore.get(idempotencyKey).then((existing) => {
    if (existing) {
      // Duplicate bid - return the original result
      request.message = existing;
      return request.ok();
    }

    // Process the bid (validation logic here)
    return processAndValidateBid(request, message).then((result) => {
      // Store result with TTL for idempotency
      return kvstore.set(idempotencyKey, result).then(() => {
        request.message = result;
        return request.ok();
      });
    });
  });
};
```

## Minimum Bid Increment Enforcement

### Increment Configuration

| Auction Current Price | Minimum Increment |
|----------------------|-------------------|
| $0 - $99 | $1.00 |
| $100 - $499 | $5.00 |
| $500 - $999 | $10.00 |
| $1,000 - $4,999 | $25.00 |
| $5,000 - $24,999 | $50.00 |
| $25,000+ | $100.00 |

### Tiered Increment Logic

```javascript
function getMinimumIncrement(currentPrice) {
  if (currentPrice < 100) return 1.00;
  if (currentPrice < 500) return 5.00;
  if (currentPrice < 1000) return 10.00;
  if (currentPrice < 5000) return 25.00;
  if (currentPrice < 25000) return 50.00;
  return 100.00;
}

// Use in PubNub Function validation
const minimumIncrement = getMinimumIncrement(currentBid);
const minimumBid = currentBid + minimumIncrement;
```

## Outbid Notifications

### Server-Side Notification Publishing

```javascript
// Inside PubNub Function after accepting a bid
function sendOutbidNotification(pubnub, previousBidderId, auctionId, details) {
  if (!previousBidderId) return Promise.resolve();

  return pubnub.publish({
    channel: `user.${previousBidderId}.notifications`,
    message: {
      type: 'bid_outbid',
      auctionId: auctionId,
      auctionTitle: details.title,
      yourBid: details.previousBid,
      newHighBid: details.newBid,
      minimumToRegain: details.newBid + details.minimumIncrement,
      timestamp: Date.now()
    }
  });
}
```

### Client-Side Notification Handling

```javascript
// Subscribe to personal notification channel
pubnub.subscribe({
  channels: [`user.${pubnub.getUserId()}.notifications`]
});

pubnub.addListener({
  message: (event) => {
    const msg = event.message;

    switch (msg.type) {
      case 'bid_outbid':
        showNotification({
          title: 'You have been outbid!',
          body: `${msg.auctionTitle}: New high bid is $${msg.newHighBid}. Bid at least $${msg.minimumToRegain} to retake the lead.`,
          action: { label: 'Bid Now', auctionId: msg.auctionId }
        });
        break;

      case 'auction_won':
        showNotification({
          title: 'Congratulations! You won!',
          body: `You won "${msg.auctionTitle}" with a bid of $${msg.amount}.`,
          action: { label: 'View Details', auctionId: msg.auctionId }
        });
        break;

      case 'auction_ending_soon':
        showNotification({
          title: 'Auction ending soon',
          body: `"${msg.auctionTitle}" ends in ${msg.remainingMinutes} minutes.`,
          action: { label: 'View Auction', auctionId: msg.auctionId }
        });
        break;
    }
  }
});
```

## Bid History and Audit Trail

### Fetching Bid History

```javascript
async function getBidHistory(pubnub, auctionId, count = 100) {
  const response = await pubnub.fetchMessages({
    channels: [`auction.${auctionId}`],
    count: count,
    includeMessageActions: true
  });

  const messages = response.channels[`auction.${auctionId}`] || [];

  // Filter to accepted bids only
  const bids = messages
    .filter(msg => msg.message.type === 'bid_accepted')
    .map(msg => ({
      bidderId: msg.message.bidderId,
      amount: msg.message.amount,
      bidNumber: msg.message.bidNumber,
      timetoken: msg.timetoken,
      timestamp: new Date(parseInt(msg.timetoken) / 10000)
    }));

  return bids;
}
```

### Bid Activity Feed

```javascript
// Publish activity events for the bid feed
async function publishBidActivity(pubnub, auctionId, bid) {
  await pubnub.publish({
    channel: `auction.${auctionId}.activity`,
    message: {
      type: 'bid_activity',
      bidderDisplayName: maskBidderId(bid.bidderId),
      amount: bid.amount,
      bidNumber: bid.bidNumber,
      timestamp: Date.now()
    },
    storeInHistory: true
  });
}

// Mask bidder identity for public display
function maskBidderId(bidderId) {
  if (bidderId.length <= 4) return '****';
  return bidderId.substring(0, 2) + '***' + bidderId.slice(-2);
}
```

### React Bid History Component

```javascript
function BidHistory({ pubnub, auctionId }) {
  const [bids, setBids] = useState([]);

  useEffect(() => {
    // Load historical bids
    getBidHistory(pubnub, auctionId).then(setBids);

    // Listen for new bids
    const listener = {
      message: (event) => {
        if (event.channel === `auction.${auctionId}` &&
            event.message.type === 'bid_accepted') {
          setBids(prev => [...prev, {
            bidderId: event.message.bidderId,
            amount: event.message.amount,
            bidNumber: event.message.bidNumber,
            timestamp: new Date()
          }]);
        }
      }
    };

    pubnub.addListener(listener);
    return () => pubnub.removeListener(listener);
  }, [pubnub, auctionId]);

  return (
    <div className="bid-history">
      <h3>Bid History ({bids.length} bids)</h3>
      <ul>
        {bids.slice().reverse().map((bid, i) => (
          <li key={bid.bidNumber || i}>
            <span className="bidder">{maskBidderId(bid.bidderId)}</span>
            <span className="amount">${bid.amount.toFixed(2)}</span>
            <span className="time">
              {bid.timestamp.toLocaleTimeString()}
            </span>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

## Error Handling

### Common Bid Errors

| Error | Cause | Client Action |
|-------|-------|---------------|
| `Bid must be at least X` | Amount below minimum | Show updated minimum, prompt re-bid |
| `Auction is completed` | Auction has ended | Disable bid button, show result |
| `Auction is paused` | Admin paused the auction | Show paused state, wait for resume |
| `You are already the high bidder` | Bidder already leads | Show confirmation, no action needed |
| `Bid processing in progress` | Race condition lock | Auto-retry after short delay |
| `Auction not found` | Invalid auction ID | Show error, redirect to catalog |

### Client Error Handler

```javascript
async function handleBidSubmission(pubnub, auctionId, amount) {
  try {
    const result = await placeBidWithRetry(pubnub, auctionId, amount);
    return { success: true, bid: result };
  } catch (error) {
    if (error instanceof BidRejectedError) {
      const details = error.details;

      if (details.reason.includes('at least')) {
        return {
          success: false,
          error: 'bid_too_low',
          message: `Minimum bid is $${details.minimumBid.toFixed(2)}`,
          suggestedAmount: details.minimumBid
        };
      }

      if (details.reason.includes('completed') || details.reason.includes('ended')) {
        return {
          success: false,
          error: 'auction_ended',
          message: 'This auction has ended'
        };
      }

      if (details.reason.includes('already the high bidder')) {
        return {
          success: false,
          error: 'already_winning',
          message: 'You are already the highest bidder'
        };
      }

      return {
        success: false,
        error: 'bid_rejected',
        message: details.reason
      };
    }

    return {
      success: false,
      error: 'network_error',
      message: 'Failed to submit bid. Please check your connection and try again.'
    };
  }
}
```

## Best Practices

1. **Always validate server-side** - Client-side validation is for UX only. PubNub Functions must be the authority on bid acceptance.
2. **Use idempotency keys** - Generate a unique key per bid attempt to prevent duplicate processing from network retries.
3. **Implement KV Store locking** - Use compare-and-set patterns with short-lived locks to prevent race conditions between simultaneous bids.
4. **Mask bidder identities** - In public bid feeds, mask bidder IDs to protect privacy while still showing distinct participants.
5. **Store all bids in history** - Use `storeInHistory: true` for all accepted bids and auction state changes for dispute resolution.
6. **Send outbid notifications immediately** - Use PubNub Functions to publish outbid notifications in the same transaction as bid acceptance.
7. **Handle network failures gracefully** - Implement retry with exponential backoff and show clear error states to the user.
8. **Enforce tiered increments** - Use price-based minimum increments to prevent micro-bid spam at higher price points.
9. **Log rejected bids** - Store rejected bid attempts for fraud detection and analytics purposes.
10. **Time-box lock TTLs** - Keep KV Store locks short (2-3 seconds) with automatic expiration to prevent deadlocks.
