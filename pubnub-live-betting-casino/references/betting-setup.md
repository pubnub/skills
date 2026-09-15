# PubNub Betting Platform Setup

## Overview

Market channel hierarchy, odds publish, market suspension, Access Manager, and encryption for wager channels. Pull SDK init from MCP / docs; do not copy install manuals or odds-format math here.

Enable **Access Manager**, **Message Persistence** (bet audit), and **Functions** (wager validation) on the keyset. Initialize with `setToken` plus CryptoModule — retrieve current SDK parameters via MCP.

## Market channel design

| Channel Pattern | Purpose | Example |
|----------------|---------|---------|
| `event.{sport}.{eventId}` | Event-level updates (scores, status) | `event.football.12345` |
| `event.{sport}.{eventId}.market.{marketId}` | Market-level odds updates | `event.football.12345.market.match-winner` |
| `odds.{sport}.live` | Aggregated live odds feed per sport | `odds.football.live` |
| `wagers.submit` | Wager submission channel | `wagers.submit` |
| `wagers.{userId}.status` | Per-user bet status updates | `wagers.user-789.status` |
| `balance.{userId}` | Per-user balance updates | `balance.user-789` |

### Channel groups

```javascript
await pubnub.channelGroups.addChannels({
  channelGroup: 'event-football-12345-markets',
  channels: [
    'event.football.12345.market.match-winner',
    'event.football.12345.market.over-under-2-5',
    'event.football.12345.market.both-teams-score',
    'event.football.12345.market.correct-score'
  ]
});

pubnub.subscribe({
  channelGroups: ['event-football-12345-markets']
});
```

## Odds broadcasting

Publish decimal/fractional/American as operator-supplied fields on the market channel. Do not convert formats in this skill.

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
        odds: sel.odds,
        movement: sel.movement,
        status: sel.status
      })),
      suspended: market.suspended,
      inPlay: market.inPlay,
      timestamp: Date.now(),
      sequence: market.sequenceNumber
    }
  });
}
```

### Market suspension

Suspend before recalculating odds (goals, cards, VAR). Same sequence as [betting-patterns.md](betting-patterns.md) incident handler.

```javascript
async function suspendMarket(eventId, marketId, reason) {
  await pubnub.publish({
    channel: `event.football.${eventId}.market.${marketId}`,
    message: {
      type: 'market_suspension',
      marketId: marketId,
      suspended: true,
      reason: reason,
      timestamp: Date.now()
    }
  });
}

async function resumeMarket(eventId, market) {
  market.suspended = false;
  await publishOdds(eventId, market);
}
```

## Access Manager

Odds engines write market channels; clients read markets and write `wagers.submit` only.

```javascript
const oddsToken = await pubnub.grantToken({
  ttl: 60,
  authorized_uuid: 'odds-engine',
  patterns: {
    channels: {
      '^event\\.football\\..*\\.market\\..*$': { write: true }
    }
  }
});

const userToken = await pubnub.grantToken({
  ttl: 1440,
  authorized_uuid: userId,
  resources: {
    channels: {
      'wagers.submit': { write: true },
      [`wagers.${userId}.status`]: { read: true },
      [`balance.${userId}`]: { read: true }
    }
  },
  patterns: {
    channels: {
      '^event\\.football\\..*\\.market\\..*$': { read: true }
    }
  }
});
```

Encrypt wager and balance channels with CryptoModule (see [pubnub-security encryption](../../pubnub-security/references/encryption.md)).

## Regulatory enforcement (PubNub)

Enforce via AM grants + Message Persistence + Before-Publish (self-exclusion / geo in [betting-patterns.md](betting-patterns.md)). Do not copy jurisdiction retention or age-verify primers here.

## Error handling

Wire status categories per [dropped-connections](../../pubnub-presence/references/dropped-connections.md) and [backoff-and-jitter](../../pubnub-reliability/references/backoff-and-jitter.md). **Betting-specific on reconnect:** refresh auth token on `PNAccessDeniedCategory`; refresh all odds on `PNReconnectedCategory`; show stale-odds warning on `PNNetworkIssuesCategory`.

## Best Practices

1. **Channel groups** for market subscriptions instead of per-market subscribe calls.
2. **Sequence numbers** on odds messages for out-of-order delivery.
3. **Encrypt** wager and balance channels.
4. **Message Persistence** on odds and wager channels for audit.
5. **Short AM TTLs**; clients cannot write odds channels.
6. **Presence** on market/table channels for occupancy, not as a compliance product.
7. **Dot-separated channel hierarchies** for AM pattern matching.
