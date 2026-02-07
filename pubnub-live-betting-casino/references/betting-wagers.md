# PubNub Wager Management

## Overview

This reference covers wager submission, server-side validation using PubNub Functions, odds price locking, bet settlement, cash-out calculations, and balance management for real-time betting platforms.

## Wager Lifecycle

| State | Description | Channel |
|-------|-------------|---------|
| `pending` | Bet submitted, awaiting validation | `wagers.submit` |
| `accepted` | Validated and accepted | `wagers.{userId}.status` |
| `rejected` | Failed validation | `wagers.{userId}.status` |
| `live` | Active on an in-play event | `wagers.{userId}.status` |
| `won` | Selection resulted as winner | `wagers.{userId}.status` |
| `lost` | Selection resulted as loser | `wagers.{userId}.status` |
| `void` | Market voided (dead heat, abandonment) | `wagers.{userId}.status` |
| `cashed_out` | User accepted a cash-out offer | `wagers.{userId}.status` |

## Bet Types

| Bet Type | Description | Payout Calculation |
|----------|-------------|-------------------|
| Single | One selection | `stake * odds` |
| Double | Two selections, both must win | `stake * odds1 * odds2` |
| Treble | Three selections, all must win | `stake * odds1 * odds2 * odds3` |
| Accumulator | Four or more selections | `stake * odds1 * ... * oddsN` |
| Each Way | Win and place components | `(stake * winOdds) + (stake * placeOdds)` |

## Submitting a Wager

### Client-Side Bet Slip

```javascript
class BetSlip {
  constructor(pubnub, userId) {
    this.pubnub = pubnub;
    this.userId = userId;
    this.selections = [];
    this.stake = 0;
    this.betType = 'single';
  }

  addSelection(selection) {
    if (this.selections.find(s => s.eventId === selection.eventId)) {
      throw new Error('Cannot add multiple selections from the same event');
    }
    this.selections.push({
      eventId: selection.eventId,
      marketId: selection.marketId,
      selectionId: selection.selectionId,
      selectionName: selection.selectionName,
      oddsAtSelection: selection.currentOdds,
      timestamp: Date.now()
    });
  }

  calculatePotentialReturn() {
    const combinedOdds = this.selections.reduce((acc, sel) => acc * sel.oddsAtSelection, 1);
    return this.stake * combinedOdds;
  }

  async submit() {
    if (this.selections.length === 0) throw new Error('No selections');
    if (this.stake <= 0) throw new Error('No stake set');
    const betId = crypto.randomUUID();
    await this.pubnub.publish({
      channel: 'wagers.submit',
      message: { betId, userId: this.userId, betType: this.betType, selections: this.selections, stake: this.stake, currency: 'USD', potentialReturn: this.calculatePotentialReturn(), timestamp: Date.now() }
    });
    return betId;
  }
}
```

## Server-Side Validation with PubNub Functions

### Before Publish Handler

```javascript
// PubNub Function: Before Publish on 'wagers.submit'
export default (request) => {
  const message = request.message;

  if (!message.betId || !message.userId || !message.selections || !message.stake) {
    request.message = { type: 'bet_rejected', betId: message.betId, reason: 'Missing required fields', code: 'INVALID_PAYLOAD' };
    return request.abort();
  }

  if (message.stake < 0.50 || message.stake > 10000) {
    request.message = { type: 'bet_rejected', betId: message.betId, reason: 'Stake out of range', code: 'INVALID_STAKE' };
    return request.abort();
  }

  if (message.selections.length > 20) {
    request.message = { type: 'bet_rejected', betId: message.betId, reason: 'Maximum 20 selections', code: 'TOO_MANY_SELECTIONS' };
    return request.abort();
  }

  message.serverTimestamp = Date.now();
  message.status = 'pending';
  return request.ok();
};
```

### Odds Verification Function

```javascript
// PubNub Function: After Publish on 'wagers.submit'
const db = require('kvstore');
const pubnub = require('pubnub');

export default async (request) => {
  const bet = request.message;
  const DRIFT_THRESHOLD = 0.05;

  for (const selection of bet.selections) {
    const storedOdds = await db.get(`odds:${selection.eventId}:${selection.marketId}:${selection.selectionId}`);

    if (!storedOdds) {
      await pubnub.publish({ channel: `wagers.${bet.userId}.status`, message: { type: 'bet_rejected', betId: bet.betId, reason: 'Market not available', code: 'MARKET_NOT_FOUND' } });
      return request.ok();
    }

    const drift = Math.abs(parseFloat(storedOdds.decimal) - selection.oddsAtSelection) / selection.oddsAtSelection;
    if (drift > DRIFT_THRESHOLD) {
      await pubnub.publish({ channel: `wagers.${bet.userId}.status`, message: { type: 'odds_changed', betId: bet.betId, selection: selection.selectionId, submittedOdds: selection.oddsAtSelection, currentOdds: parseFloat(storedOdds.decimal), code: 'ODDS_DRIFT' } });
      return request.ok();
    }
  }

  await pubnub.publish({ channel: `wagers.${bet.userId}.status`, message: { type: 'bet_accepted', betId: bet.betId, selections: bet.selections, stake: bet.stake, potentialReturn: bet.potentialReturn, timestamp: Date.now() } });
  return request.ok();
};
```

## Balance Management

```javascript
const db = require('kvstore');

async function checkAndReserveBalance(userId, stake, currency, betId) {
  const balanceKey = `balance:${userId}:${currency}`;
  const balance = await db.get(balanceKey);

  if (!balance || parseFloat(balance.available) < stake) {
    return { valid: false, available: balance ? balance.available : 0 };
  }

  const newAvailable = parseFloat(balance.available) - stake;
  const newReserved = parseFloat(balance.reserved || 0) + stake;

  await db.set(balanceKey, { available: newAvailable.toFixed(2), reserved: newReserved.toFixed(2), lastBetId: betId, updatedAt: Date.now() });
  return { valid: true, available: newAvailable, reserved: newReserved };
}
```

### Client-Side Balance Listener

```javascript
pubnub.subscribe({ channels: [`balance.${userId}`] });

pubnub.addListener({
  message: (event) => {
    if (event.channel === `balance.${userId}`) {
      updateBalanceDisplay(event.message.available, event.message.currency);
    }
  }
});
```

## Bet Settlement

```javascript
async function settleBet(pubnub, db, bet, result) {
  const balanceKey = `balance:${bet.userId}:${bet.currency}`;
  const balance = await db.get(balanceKey);
  let payout = 0;
  let status = 'lost';

  if (result === 'won') { payout = bet.potentialReturn; status = 'won'; }
  else if (result === 'void') { payout = bet.stake; status = 'void'; }

  const newAvailable = parseFloat(balance.available) + payout;
  const newReserved = parseFloat(balance.reserved) - bet.stake;

  await db.set(balanceKey, { available: newAvailable.toFixed(2), reserved: newReserved.toFixed(2), updatedAt: Date.now() });

  await pubnub.publish({
    channel: `wagers.${bet.userId}.status`,
    message: { type: 'bet_settled', betId: bet.betId, status, stake: bet.stake, payout, settledAt: Date.now() }
  });
  await pubnub.publish({
    channel: `balance.${bet.userId}`,
    message: { type: 'balance_update', available: newAvailable, reserved: newReserved, currency: bet.currency, reason: 'bet_settled', timestamp: Date.now() }
  });
}
```

### Accumulator Settlement

```javascript
async function settleAccumulator(pubnub, db, bet, results) {
  let allWon = true;
  let adjustedOdds = 1;

  for (const selection of bet.selections) {
    const result = results[selection.selectionId];
    if (result === 'lost') { allWon = false; break; }
    else if (result === 'void') { adjustedOdds *= 1.0; }
    else { adjustedOdds *= selection.oddsAtSelection; }
  }

  const payout = allWon ? bet.stake * adjustedOdds : 0;
  await settleBet(pubnub, db, { ...bet, potentialReturn: payout }, allWon ? 'won' : 'lost');
}
```

## Cash-Out

### Cash-Out Calculation

```javascript
function calculateCashOut(bet, currentOdds) {
  const MARGIN = 0.95;

  if (bet.betType === 'single') {
    const current = currentOdds[bet.selections[0].selectionId];
    if (!current || current <= 1.0) return null;
    return Math.round(bet.stake * (bet.selections[0].oddsAtSelection / current) * MARGIN * 100) / 100;
  }

  let value = bet.stake;
  for (const sel of bet.selections) {
    const current = currentOdds[sel.selectionId];
    if (!current) return null;
    value *= sel.resulted === 'won' ? sel.oddsAtSelection : (sel.oddsAtSelection / current);
  }
  return Math.round(value * MARGIN * 100) / 100;
}
```

### Cash-Out Offer Broadcasting

```javascript
async function broadcastCashOutOffers(pubnub, db, userId) {
  const activeBets = await db.get(`active_bets:${userId}`);
  if (!activeBets) return;

  for (const bet of activeBets) {
    const currentOdds = await getCurrentOdds(db, bet.selections);
    const cashOutValue = calculateCashOut(bet, currentOdds);
    if (cashOutValue && cashOutValue > 0) {
      await pubnub.publish({
        channel: `wagers.${userId}.status`,
        message: { type: 'cashout_offer', betId: bet.betId, cashOutValue, originalStake: bet.stake, expiresAt: Date.now() + 10000, timestamp: Date.now() }
      });
    }
  }
}
```

## Odds Movement Protection

```javascript
function lockPrice(selection, currentOdds) {
  return { ...selection, oddsAtSelection: currentOdds, lockedAt: Date.now(), lockExpiry: Date.now() + 30000 };
}

function validatePriceLock(selection) {
  return Date.now() > selection.lockExpiry
    ? { valid: false, reason: 'Price lock expired' }
    : { valid: true };
}
```

## Error Handling for Rejected Bets

| Code | Description | User Action |
|------|-------------|-------------|
| `INVALID_PAYLOAD` | Missing or malformed bet data | Fix bet slip and resubmit |
| `INVALID_STAKE` | Stake below min or above max | Adjust stake amount |
| `INSUFFICIENT_FUNDS` | Not enough balance | Deposit funds |
| `MARKET_SUSPENDED` | Market currently suspended | Wait for market to reopen |
| `MARKET_NOT_FOUND` | Market no longer exists | Remove selection |
| `ODDS_DRIFT` | Odds changed beyond threshold | Accept new odds or cancel |
| `SELF_EXCLUDED` | User is self-excluded | Contact support |
| `GEO_RESTRICTED` | User location not permitted | Inform user of restriction |

### Client-Side Error Handler

```javascript
pubnub.addListener({
  message: (event) => {
    if (event.channel !== `wagers.${userId}.status`) return;
    const msg = event.message;
    switch (msg.type) {
      case 'bet_accepted': showSuccess(`Bet accepted!`); clearBetSlip(); break;
      case 'bet_rejected': showError(`Rejected: ${msg.reason}`); break;
      case 'odds_changed': showOddsChangeDialog(msg.selection, msg.submittedOdds, msg.currentOdds); break;
      case 'bet_settled': if (msg.status === 'won') showWinNotification(msg.betId, msg.payout); break;
      case 'cashout_offer': showCashOutOffer(msg.betId, msg.cashOutValue); break;
    }
  }
});
```

## Python Settlement Example

```python
from pubnub.pn_configuration import PNConfiguration
from pubnub.pubnub import PubNub
import time

config = PNConfiguration()
config.subscribe_key = "sub-c-..."
config.publish_key = "pub-c-..."
config.user_id = "settlement-engine"
config.cipher_key = "encryption-key"
pubnub = PubNub(config)

def settle_single_bet(bet, result):
    payout = 0
    status = "lost"
    if result == "won":
        payout = bet["stake"] * bet["selections"][0]["oddsAtSelection"]
        status = "won"
    elif result == "void":
        payout = bet["stake"]
        status = "void"

    pubnub.publish().channel(f"wagers.{bet['userId']}.status").message({
        "type": "bet_settled", "betId": bet["betId"],
        "status": status, "payout": round(payout, 2),
        "settledAt": int(time.time() * 1000)
    }).sync()
```

## Best Practices

1. **Always validate server-side** using PubNub Functions Before Publish handlers; never trust client-submitted odds
2. **Lock odds at selection time** and include a configurable expiry window (e.g., 30 seconds)
3. **Use atomic balance operations** in KV Store to prevent race conditions during concurrent bet placement
4. **Include bet IDs in all messages** to enable end-to-end tracing through submission, acceptance, and settlement
5. **Implement idempotency** by checking for duplicate bet IDs before processing
6. **Set cash-out offer expiry** to prevent stale offers from being accepted after odds movement
7. **Encrypt all wager and balance channels** to protect financial data in transit
8. **Publish settlement notifications** to both the wager status and balance channels simultaneously
9. **Log all bet state transitions** to Message Persistence for audit and dispute resolution
10. **Rate-limit wager submissions** per user to prevent automated abuse
