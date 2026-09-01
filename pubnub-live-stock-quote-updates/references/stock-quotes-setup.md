# PubNub Stock Quote Setup

This reference covers channel architecture design, SDK initialization, market data ingestion, and quote broadcasting patterns for delivering real-time stock quotes through PubNub.

## Channel Design for Market Data

### Channel Naming Conventions

Use dot-delimited namespaces to organize market data channels. This enables wildcard subscriptions and clean filtering.

| Channel Pattern | Purpose | Example |
|-----------------|---------|---------|
| `quotes.<SYMBOL>` | Per-symbol quote updates | `quotes.AAPL`, `quotes.TSLA` |
| `sector.<NAME>` | Sector-level aggregates | `sector.tech`, `sector.energy` |
| `index.<NAME>` | Index values | `index.SPX`, `index.DJI`, `index.NDX` |
| `market.status` | Market open/close events | `market.status` |
| `news.<SYMBOL>` | Per-symbol news headlines | `news.AAPL` |
| `trades.<SYMBOL>` | Individual trade executions | `trades.AAPL` |

### Wildcard Subscription Examples

```javascript
// Subscribe to all quote channels
pubnub.subscribe({ channels: ['quotes.*'] });

// Subscribe to all indices
pubnub.subscribe({ channels: ['index.*'] });
```

### Channel Groups for User Watchlists

Channel groups allow each user to maintain a dynamic watchlist without reconnecting.

```javascript
// Create a watchlist channel group for a user
await pubnub.channelGroups.addChannels({
  channelGroup: 'watchlist_user-789',
  channels: ['quotes.AAPL', 'quotes.GOOGL', 'quotes.AMZN']
});

// Subscribe to the entire watchlist
pubnub.subscribe({ channelGroups: ['watchlist_user-789'] });

// Add a symbol later without resubscribing
await pubnub.channelGroups.addChannels({
  channelGroup: 'watchlist_user-789',
  channels: ['quotes.NVDA']
});

// Remove a symbol
await pubnub.channelGroups.removeChannels({
  channelGroup: 'watchlist_user-789',
  channels: ['quotes.AMZN']
});
```

## SDK Initialization for Financial Applications

### Server-Side Publisher (Node.js)

The server-side component ingests data from market data providers and publishes to PubNub.

```javascript
import PubNub from 'pubnub';

const pubnub = new PubNub({
  publishKey: 'pub-c-...',
  subscribeKey: 'sub-c-...',
  secretKey: 'sec-c-...',  // Server-side only for Access Manager
  userId: 'market-data-publisher',
  ssl: true,
  retryConfiguration: PubNub.LinearRetryPolicy({
    delay: 2,
    maximumRetry: 5
  })
});
```

### Client-Side Subscriber (Browser / React Native)

```javascript
import PubNub from 'pubnub';

const pubnub = new PubNub({
  subscribeKey: 'sub-c-...',
  userId: `client-${userId}`,
  ssl: true,
  authKey: authTokenFromServer,  // For Access Manager gated data tiers
  restore: true,                 // Auto-reconnect and catch up on missed messages
  retryConfiguration: PubNub.LinearRetryPolicy({
    delay: 3,
    maximumRetry: 10
  })
});
```

### Python Publisher (Alternative Backend)

```python
from pubnub.pnconfiguration import PNConfiguration
from pubnub.pubnub import PubNub

pnconfig = PNConfiguration()
pnconfig.publish_key = "pub-c-..."
pnconfig.subscribe_key = "sub-c-..."
pnconfig.secret_key = "sec-c-..."
pnconfig.user_id = "market-data-publisher"
pnconfig.ssl = True

pubnub = PubNub(pnconfig)
```

## Data Ingestion from Market Data Providers

### Connecting to a Market Data Feed

```javascript
import WebSocket from 'ws';

function connectToMarketFeed(pubnub) {
  const ws = new WebSocket('wss://provider.example.com/feed', {
    headers: { 'Authorization': `Bearer ${process.env.MARKET_DATA_API_KEY}` }
  });

  ws.on('open', () => {
    ws.send(JSON.stringify({
      action: 'subscribe',
      symbols: ['AAPL', 'GOOGL', 'MSFT', 'TSLA', 'AMZN', 'NVDA']
    }));
  });

  ws.on('message', (data) => {
    const raw = JSON.parse(data);
    const normalized = normalizeQuote(raw);
    publishQuote(pubnub, normalized);
  });

  ws.on('close', () => {
    console.warn('Market feed disconnected, reconnecting in 5s...');
    setTimeout(() => connectToMarketFeed(pubnub), 5000);
  });

  ws.on('error', (err) => {
    console.error('Market feed error:', err.message);
  });
}
```

### Normalizing Provider Data

Different market data providers return data in varied formats. Normalize to a consistent schema before publishing.

```javascript
function normalizeQuote(raw) {
  return {
    symbol: raw.sym || raw.symbol || raw.ticker,
    price: parseFloat(raw.last || raw.price || raw.ltp),
    bid: parseFloat(raw.bid || raw.bidPrice || 0),
    ask: parseFloat(raw.ask || raw.askPrice || 0),
    volume: parseInt(raw.vol || raw.volume || raw.cumVol || 0, 10),
    open: parseFloat(raw.open || raw.openPrice || 0),
    high: parseFloat(raw.high || raw.dayHigh || 0),
    low: parseFloat(raw.low || raw.dayLow || 0),
    prevClose: parseFloat(raw.prevClose || raw.previousClose || 0),
    change: 0,
    changePct: 0,
    timestamp: raw.timestamp || Date.now()
  };
}

function enrichQuote(quote) {
  if (quote.prevClose > 0) {
    quote.change = parseFloat((quote.price - quote.prevClose).toFixed(4));
    quote.changePct = parseFloat(((quote.change / quote.prevClose) * 100).toFixed(4));
  }
  return quote;
}
```

## Quote Broadcasting Patterns

### Full Quote Publish

Use `publish` for comprehensive quote updates containing full market data.

```javascript
async function publishQuote(pubnub, rawQuote) {
  const quote = enrichQuote(rawQuote);
  try {
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
      meta: { sector: getSector(quote.symbol), exchange: getExchange(quote.symbol) }
    });
  } catch (err) {
    console.error(`Failed to publish quote for ${quote.symbol}:`, err.message);
  }
}
```

### Signal for High-Frequency Ticks

Use `signal` for rapid price-only updates. Signals have a 64-byte payload limit but are lower cost.

```javascript
async function publishTick(pubnub, symbol, price) {
  try {
    await pubnub.signal({
      channel: `quotes.${symbol}`,
      message: { p: price, t: Date.now() }
    });
  } catch (err) {
    console.error(`Failed to signal tick for ${symbol}:`, err.message);
  }
}
```

### Comparison: Publish vs Signal

| Feature | Publish | Signal |
|---------|---------|--------|
| Max payload | 32 KB | 64 bytes |
| History storage | Yes (optional) | No |
| Message cost | Standard | Reduced |
| Use case | Full quote snapshots | Price-only ticks |
| Rate limit | Standard publish rate | Higher throughput allowed |
| Triggers Functions | Yes | Yes |

## Throttling and Batching

High-frequency feeds can produce hundreds of updates per second for popular symbols. Throttle to avoid unnecessary load.

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

async function batchPublishQuotes(pubnub, quotes) {
  const BATCH_SIZE = 20;
  const BATCH_DELAY_MS = 100;

  for (let i = 0; i < quotes.length; i += BATCH_SIZE) {
    const batch = quotes.slice(i, i + BATCH_SIZE);
    await Promise.allSettled(batch.map((q) => publishQuote(pubnub, q)));
    if (i + BATCH_SIZE < quotes.length) {
      await new Promise((r) => setTimeout(r, BATCH_DELAY_MS));
    }
  }
}
```

## Update Frequency Guidelines

| Data Type | Recommended Frequency | Method | Notes |
|-----------|----------------------|--------|-------|
| Last price tick | 100-500ms | Signal | Price-only, minimal payload |
| Full quote snapshot | 1-5 seconds | Publish | Includes bid/ask/volume |
| Index values | 1-5 seconds | Publish | SPX, DJI, NDX |
| Sector aggregates | 5-15 seconds | Publish | Computed server-side |
| Market status | On change | Publish | Open, close, halt events |
| Daily summary | End of day | Publish | OHLCV summary with history |

## Listening for Quotes on the Client

### Complete Client Setup

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
      if (event.category === 'PNConnectedCategory') callbacks.onConnected();
      if (event.category === 'PNReconnectedCategory') callbacks.onReconnected();
      if (event.category === 'PNNetworkDownCategory') callbacks.onDisconnected();
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

### React Hook Example

```javascript
import { useEffect, useState, useRef } from 'react';
import PubNub from 'pubnub';

function useStockQuotes(subscribeKey, userId, symbols) {
  const [quotes, setQuotes] = useState({});
  const pubnubRef = useRef(null);

  useEffect(() => {
    const pn = new PubNub({ subscribeKey, userId, restore: true });
    pubnubRef.current = pn;
    const channels = symbols.map((s) => `quotes.${s}`);

    pn.addListener({
      message: (event) => {
        setQuotes((prev) => ({ ...prev, [event.message.symbol]: event.message }));
      },
      signal: (event) => {
        const sym = event.channel.replace('quotes.', '');
        setQuotes((prev) => ({
          ...prev,
          [sym]: { ...prev[sym], price: event.message.p, timestamp: event.message.t }
        }));
      }
    });

    pn.subscribe({ channels });
    return () => { pn.unsubscribeAll(); pn.destroy(); };
  }, [subscribeKey, userId, JSON.stringify(symbols)]);

  return quotes;
}
```

## Best Practices

- **Separate channels per symbol** rather than multiplexing multiple symbols on a single channel. This allows clients to subscribe only to what they need and enables efficient wildcard subscriptions.
- **Use signals for sub-second price ticks** and reserve full publish for quote snapshots every 1-5 seconds. This reduces message costs while maintaining real-time feel.
- **Store full quote snapshots in history** so that clients joining mid-session can fetch the last known quote immediately with `fetchMessages()`.
- **Implement reconnection logic** on both the server ingestion side and the client subscription side. Use PubNub's `restore: true` and retry policies to handle network blips.
- **Normalize all provider data** into a consistent schema before publishing. This decouples clients from any specific market data vendor format.
- **Throttle per-symbol publish rates** to avoid exceeding rate limits during volatile markets. Use signals for intermediate ticks between full publishes.
- **Tag messages with metadata** (sector, exchange) using the `meta` field on publish. This enables server-side filtering in PubNub Functions without parsing the message body.
- **Monitor publish latency** and error rates. Set up alerts for failed publishes so data gaps are caught quickly during market hours.
- **Use Access Manager** to enforce data entitlements. Grant read access to premium channels only to authenticated, entitled users.
