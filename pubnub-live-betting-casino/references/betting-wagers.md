# PubNub Wager Management

## Overview

Wager submission channels, Before-Publish validation, balance listener, settlement/cash-out **publish** shapes. Payout formulas and bet-type math belong to the operator, not this skill.

## Wager lifecycle

| State | Description | Channel |
|-------|-------------|---------|
| `pending` | Bet submitted, awaiting validation | `wagers.submit` |
| `accepted` | Validated and accepted | `wagers.{userId}.status` |
| `rejected` | Failed validation | `wagers.{userId}.status` |
| `live` | Active on an in-play event | `wagers.{userId}.status` |
| `won` / `lost` / `void` / `cashed_out` | Terminal | `wagers.{userId}.status` |

## Submitting a wager

```javascript
async function submitWager(pubnub, userId, slip) {
  const betId = crypto.randomUUID();
  await pubnub.publish({
    channel: 'wagers.submit',
    message: {
      betId,
      userId,
      betType: slip.betType,
      selections: slip.selections,
      stake: slip.stake,
      currency: slip.currency,
      timestamp: Date.now()
    }
  });
  return betId;
}
```

## Before-Publish validation (S1)

Use the [server-authoritative Before-Publish pattern](../../pubnub-functions/references/functions-patterns.md) for `wagers.submit`. **Betting-specific checks:**

- Required fields: `betId`, `userId`, `selections`, `stake`
- Stake range and max selections per slip (jurisdiction-specific)
- Stamp `serverTimestamp` and `status: 'pending'` on accept

Odds verification and balance reservation run in After-Publish or downstream Functions.

### Odds verification (After Publish)

| Check | Betting rule |
|-------|--------------|
| Market exists | `odds:<eventId>:<marketId>:<selectionId>` in KV Store |
| Drift | Reject or prompt if stored decimal drifts beyond jurisdiction threshold vs `oddsAtSelection` |
| Accept | Publish `bet_accepted` to `wagers.{userId}.status` |

After-Publish wiring follows [functions-patterns After Publish](../../pubnub-functions/references/functions-patterns.md).

## Balance reservation and listener

Reserve stake in KV (`balance:{userId}:{currency}`) after accept: decrement `available`, increment `reserved`, stamp `lastBetId`. Clients subscribe to `balance.{userId}` for display updates — KV is source of truth.

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

## Settlement publish

After the operator computes payout, publish to both status and balance channels:

```javascript
await pubnub.publish({
  channel: `wagers.${bet.userId}.status`,
  message: { type: 'bet_settled', betId: bet.betId, status, stake: bet.stake, payout, settledAt: Date.now() }
});
await pubnub.publish({
  channel: `balance.${bet.userId}`,
  message: { type: 'balance_update', available, reserved, currency: bet.currency, reason: 'bet_settled', timestamp: Date.now() }
});
```

## Cash-out offers

Broadcast operator-computed `cashOutValue` on `wagers.{userId}.status` (`type: 'cashout_offer'`) with `expiresAt`. Do not embed cash-out formulas here.

## Price lock

Stamp `oddsAtSelection` + `lockExpiry` on the selection at slip time; Before-Publish rejects expired locks. Window length is operator-configured.

## Rejected bets

| Code | Description | Client action |
|------|-------------|---------------|
| `INVALID_PAYLOAD` | Missing or malformed bet data | Fix slip and resubmit |
| `INVALID_STAKE` | Stake below min or above max | Adjust stake |
| `INSUFFICIENT_FUNDS` | Not enough balance | Prompt deposit |
| `MARKET_SUSPENDED` | Market currently suspended | Wait for reopen |
| `MARKET_NOT_FOUND` | Market no longer exists | Remove selection |
| `ODDS_DRIFT` | Odds changed beyond threshold | Accept new odds or cancel |
| `SELF_EXCLUDED` | User is self-excluded | Contact support |
| `GEO_RESTRICTED` | User location not permitted | Inform user |

```javascript
pubnub.addListener({
  message: (event) => {
    if (event.channel !== `wagers.${userId}.status`) return;
    const msg = event.message;
    switch (msg.type) {
      case 'bet_accepted': clearBetSlip(); break;
      case 'bet_rejected': showError(msg.reason); break;
      case 'odds_changed': showOddsChangeDialog(msg); break;
      case 'bet_settled': break;
      case 'cashout_offer': showCashOutOffer(msg.betId, msg.cashOutValue); break;
    }
  }
});
```

## Best Practices

1. **Always validate server-side** (S1); never trust client-submitted odds.
2. **Lock odds at selection time** with a configurable expiry.
3. **Atomic KV balance ops** for concurrent placement.
4. **Include `betId` on every message** for tracing.
5. **Idempotency** on duplicate `betId`.
6. **Expiry on cash-out offers** so stale offers cannot be accepted.
7. **Encrypt** wager and balance channels.
8. **Publish settlement** to status and balance together.
9. **Persist state transitions** for audit.
10. **Rate-limit** `wagers.submit` per user.
