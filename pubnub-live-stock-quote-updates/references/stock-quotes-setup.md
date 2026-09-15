# PubNub Stock Quote Setup

Channel architecture, publish vs signal, throttling, and client subscribe. Provider-feed tutorials and multi-language SDK dumps are out of scope — pull init from MCP.

## Channel design

| Channel Pattern | Purpose | Example |
|-----------------|---------|---------|
| `quotes.<SYMBOL>` | Per-symbol quote updates | `quotes.AAPL`, `quotes.TSLA` |
| `sector.<NAME>` | Sector-level aggregates | `sector.tech`, `sector.energy` |
| `index.<NAME>` | Index values | `index.SPX`, `index.DJI`, `index.NDX` |
| `market.status` | Market open/close events | `market.status` |
| `news.<SYMBOL>` | Per-symbol news headlines | `news.AAPL` |
| `trades.<SYMBOL>` | Individual trade executions | `trades.AAPL` |

```javascript
pubnub.subscribe({ channels: ['quotes.*'] });
pubnub.subscribe({ channels: ['index.*'] });
```

### Channel groups for watchlists

```javascript
await pubnub.channelGroups.addChannels({
  channelGroup: 'watchlist_user-789',
  channels: ['quotes.AAPL', 'quotes.GOOGL', 'quotes.AMZN']
});

pubnub.subscribe({ channelGroups: ['watchlist_user-789'] });

await pubnub.channelGroups.addChannels({
  channelGroup: 'watchlist_user-789',
  channels: ['quotes.NVDA']
});

await pubnub.channelGroups.removeChannels({
  channelGroup: 'watchlist_user-789',
  channels: ['quotes.AMZN']
});
```

Server publishers use publish + secret key (AM). Clients: subscribe key + `setToken`, `restore: true`. Retrieve SDK parameters via MCP.

## Quote broadcasting

### Full quote (`publish`)

```javascript
async function publishQuote(pubnub, quote) {
  await pubnub.publish({
    channel: `quotes.${quote.symbol}`,
    message: {
      symbol: quote.symbol,
      price: quote.price,
      bid: quote.bid,
      ask: quote.ask,
      volume: quote.volume,
      open: quote.open,
      high: quote.high,
      low: quote.low,
      prevClose: quote.prevClose,
      change: quote.change,
      changePct: quote.changePct,
      timestamp: quote.timestamp
    },
    storeInHistory: true,
    meta: { sector: quote.sector, exchange: quote.exchange }
  });
}
```

### High-frequency ticks (`signal`)

Signals have a small payload limit (retrieve current limit via MCP / docs) and are not stored in history.

```javascript
async function publishTick(pubnub, symbol, price) {
  await pubnub.signal({
    channel: `quotes.${symbol}`,
    message: { p: price, t: Date.now() }
  });
}
```

| Feature | Publish | Signal |
|---------|---------|--------|
| History | Yes (optional) | No |
| Cost | Standard | Reduced |
| Use case | Full quote snapshots | Price-only ticks |
| Triggers Functions | Yes | Yes |

Normalize vendor ticks **before** publish so clients see one schema. Do not copy provider WebSocket connect tutorials here.

## Throttling

Stay within PubNub publish rates during volatile sessions.

```javascript
const lastPublished = new Map();
const THROTTLE_MS = 250;

function throttledPublish(pubnub, quote) {
  const now = Date.now();
  const lastTime = lastPublished.get(quote.symbol) || 0;

  if (now - lastTime >= THROTTLE_MS) {
    lastPublished.set(quote.symbol, now);
    publishQuote(pubnub, quote);
  } else {
    publishTick(pubnub, quote.symbol, quote.price);
  }
}
```

| Data Type | Recommended Frequency | Method |
|-----------|----------------------|--------|
| Last price tick | 100-500ms | Signal |
| Full quote snapshot | 1-5 seconds | Publish |
| Index / sector | 1-15 seconds | Publish |
| Market status | On change | Publish |

## Client subscribe

```javascript
function initQuoteSubscriber(pubnub, symbols, callbacks) {
  const channels = symbols.map((s) => `quotes.${s}`);

  pubnub.addListener({
    message: (event) => callbacks.onQuote(event.message),
    signal: (event) => {
      const symbol = event.channel.replace('quotes.', '');
      callbacks.onTick(symbol, event.message.p, event.message.t);
    },
    status: (event) => {
      if (event.category === 'PNReconnectedCategory') callbacks.onReconnected();
      if (event.category === 'PNNetworkDownCategory') callbacks.onDisconnected();
      if (event.category === 'PNConnectedCategory') callbacks.onConnected();
    }
  });

  pubnub.subscribe({ channels });

  return {
    addSymbol: (sym) => pubnub.subscribe({ channels: [`quotes.${sym}`] }),
    removeSymbol: (sym) => pubnub.unsubscribe({ channels: [`quotes.${sym}`] }),
    disconnect: () => pubnub.unsubscribeAll()
  };
}
```

On reconnect, refresh quote state ([stock-quotes-portfolio.md](stock-quotes-portfolio.md) `fetchMessages`). Status wiring: [dropped-connections](../../pubnub-presence/references/dropped-connections.md).

## Best Practices

- **One channel per symbol** (wildcard + channel groups).
- **Signals for sub-second ticks**; `publish` for snapshots every 1-5s.
- **`storeInHistory` on snapshots** so joiners can `fetchMessages()`.
- **`restore: true`** plus retry policy on clients.
- **Throttle per symbol**; tag `meta` (sector, exchange) for Functions filters.
- **AM** for delayed vs real-time prefixes ([stock-quotes-patterns.md](stock-quotes-patterns.md)).
