# PubNub Stock Quote Patterns

Entitlements, multi-exchange subscribe, and reconnect. Ticker/chart UI, market-hours calendars, and disclaimer copy are out of scope.

## Data entitlements (AM → channel prefix)

| Tier | Data | Delay | Channel Prefix |
|------|------|-------|----------------|
| Free | Basic quotes | 15-minute delay | `delayed.` |
| Standard | Real-time quotes | None | `quotes.` |
| Premium | Real-time + Level 2 | None | `premium.` |
| Professional | Full depth, trades | None | `pro.` |

```javascript
async function grantAccess(pubnub, userId, tier) {
  const channelPatterns = {
    free: ['delayed.*'],
    standard: ['delayed.*', 'quotes.*', 'index.*'],
    premium: ['delayed.*', 'quotes.*', 'index.*', 'premium.*'],
    professional: ['delayed.*', 'quotes.*', 'index.*', 'premium.*', 'pro.*']
  };
  const channels = channelPatterns[tier] || channelPatterns.free;

  return pubnub.grantToken({
    ttl: 60,
    authorizedUuid: userId,
    resources: {
      channels: channels.reduce((acc, ch) => {
        acc[ch] = { read: true };
        return acc;
      }, {}),
      groups: { [`watchlist_${userId}`]: { read: true, manage: true } }
    }
  });
}

function getChannelForTier(symbol, tier) {
  if (tier === 'free') return `delayed.${symbol}`;
  if (tier === 'professional') return `pro.${symbol}`;
  return `quotes.${symbol}`;
}
```

Enforce tiers with Access Manager only — never client-side checks.

## Multi-exchange subscribe

| Exchange | Channel Convention |
|----------|-------------------|
| NYSE / NASDAQ | `quotes.<SYMBOL>` |
| LSE | `quotes.LON.<SYMBOL>` |
| TSE | `quotes.TYO.<SYMBOL>` |
| HKEX | `quotes.HKG.<SYMBOL>` |

```javascript
function subscribeToExchanges(pubnub, portfolio) {
  const channelMap = {
    'NYSE': (s) => `quotes.${s}`,
    'NASDAQ': (s) => `quotes.${s}`,
    'LSE': (s) => `quotes.LON.${s}`,
    'TSE': (s) => `quotes.TYO.${s}`,
    'HKEX': (s) => `quotes.HKG.${s}`
  };
  const channels = portfolio.map((item) => {
    const resolver = channelMap[item.exchange] || channelMap['NYSE'];
    return resolver(item.symbol);
  });
  pubnub.subscribe({ channels });
  return channels;
}
```

Session/holiday calendars and delayed-quote legal copy belong to the market-data vendor agreement, not this skill. Use separate channel suffixes (e.g. `.pre` / `.post`) only if the operator splits extended-hours feeds.

## Error / reconnect

```javascript
function setupErrorHandling(pubnub, onError) {
  pubnub.addListener({
    status: (event) => {
      switch (event.category) {
        case 'PNNetworkDownCategory':
          onError({ type: 'network', message: 'Connection lost. Quotes may be stale.', recoverable: true });
          break;
        case 'PNAccessDeniedCategory':
          onError({ type: 'entitlement', message: 'Access denied. Subscription may have expired.', recoverable: false });
          break;
        case 'PNTimeoutCategory':
          onError({ type: 'timeout', message: 'Request timed out. Retrying...', recoverable: true });
          break;
        case 'PNReconnectedCategory':
          onError({ type: 'recovery', message: 'Connection restored.', recoverable: true });
          break;
      }
    }
  });
}
```

On reconnect, refresh last quotes via `fetchMessages` ([stock-quotes-portfolio.md](stock-quotes-portfolio.md)). Drop quotes with invalid `price` or crossed `bid`/`ask` before applying to UI.

## Best Practices

- **AM for data tiers.** Never rely on client-side entitlement checks.
- **Separate channels for extended-hours** if users opt in/out of those feeds.
- **Validate incoming quotes** before applying (price, bid/ask, timestamp).
- **Wildcard / channel-group subscribe** rather than per-symbol reconnects.
