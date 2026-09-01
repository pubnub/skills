# PubNub Portfolio Tracking

This reference covers watchlist management, real-time portfolio value calculation, price alert systems, gain/loss tracking, and historical data integration for building portfolio tracking applications with PubNub.

## Watchlist Implementation with Channel Groups

### Creating a Watchlist

Channel groups are the foundation for user watchlists. Each user gets a dedicated channel group that maps to their subscribed symbols.

```javascript
import PubNub from 'pubnub';

class Watchlist {
  constructor(pubnub, userId) {
    this.pubnub = pubnub;
    this.userId = userId;
    this.groupName = `watchlist_${userId}`;
    this.symbols = new Set();
  }

  async addSymbol(symbol) {
    await this.pubnub.channelGroups.addChannels({
      channelGroup: this.groupName,
      channels: [`quotes.${symbol.toUpperCase()}`]
    });
    this.symbols.add(symbol.toUpperCase());
  }

  async removeSymbol(symbol) {
    await this.pubnub.channelGroups.removeChannels({
      channelGroup: this.groupName,
      channels: [`quotes.${symbol.toUpperCase()}`]
    });
    this.symbols.delete(symbol.toUpperCase());
  }

  async addMultiple(symbols) {
    const channels = symbols.map((s) => `quotes.${s.toUpperCase()}`);
    await this.pubnub.channelGroups.addChannels({
      channelGroup: this.groupName,
      channels
    });
    symbols.forEach((s) => this.symbols.add(s.toUpperCase()));
  }

  async listSymbols() {
    const result = await this.pubnub.channelGroups.listChannels({
      channelGroup: this.groupName
    });
    return result.channels.map((ch) => ch.replace('quotes.', ''));
  }

  subscribe(onQuote, onTick) {
    this.pubnub.addListener({
      message: (event) => onQuote(event.message),
      signal: (event) => {
        const symbol = event.channel.replace('quotes.', '');
        onTick(symbol, event.message.p, event.message.t);
      }
    });
    this.pubnub.subscribe({ channelGroups: [this.groupName] });
  }

  unsubscribe() {
    this.pubnub.unsubscribe({ channelGroups: [this.groupName] });
  }
}
```

### Initializing a Watchlist on App Load

```javascript
async function initWatchlist(pubnub, userId, savedSymbols) {
  const watchlist = new Watchlist(pubnub, userId);
  const currentSymbols = await watchlist.listSymbols();

  const toAdd = savedSymbols.filter((s) => !currentSymbols.includes(s));
  const toRemove = currentSymbols.filter((s) => !savedSymbols.includes(s));
  if (toAdd.length > 0) await watchlist.addMultiple(toAdd);
  for (const s of toRemove) await watchlist.removeSymbol(s);

  const lastQuotes = await fetchLastQuotes(pubnub, savedSymbols);
  return { watchlist, lastQuotes };
}

async function fetchLastQuotes(pubnub, symbols) {
  const channels = symbols.map((s) => `quotes.${s}`);
  const result = await pubnub.fetchMessages({ channels, count: 1 });
  const quotes = {};
  for (const [channel, messages] of Object.entries(result.channels || {})) {
    if (messages.length > 0) {
      quotes[channel.replace('quotes.', '')] = messages[0].message;
    }
  }
  return quotes;
}
```

## Real-Time Portfolio Value Calculation

### Portfolio Data Model

| Field | Type | Description |
|-------|------|-------------|
| `symbol` | string | Ticker symbol |
| `shares` | number | Number of shares held |
| `avgCost` | number | Average cost basis per share |
| `currentPrice` | number | Latest market price |
| `marketValue` | number | `shares * currentPrice` |
| `totalCost` | number | `shares * avgCost` |
| `gainLoss` | number | `marketValue - totalCost` |
| `gainLossPct` | number | `(gainLoss / totalCost) * 100` |
| `dayChange` | number | `shares * quote.change` |

### Portfolio Tracker Class

```javascript
class PortfolioTracker {
  constructor(positions) {
    this.positions = new Map();
    positions.forEach((pos) => {
      this.positions.set(pos.symbol, {
        shares: pos.shares,
        avgCost: pos.avgCost,
        currentPrice: 0,
        prevClose: 0,
        lastUpdated: null
      });
    });
  }

  updatePrice(symbol, quote) {
    const pos = this.positions.get(symbol);
    if (!pos) return null;
    pos.currentPrice = quote.price;
    pos.prevClose = quote.prevClose || pos.prevClose;
    pos.lastUpdated = quote.timestamp || Date.now();
    return this.getPositionSummary(symbol);
  }

  getPositionSummary(symbol) {
    const pos = this.positions.get(symbol);
    if (!pos || pos.currentPrice === 0) return null;

    const marketValue = pos.shares * pos.currentPrice;
    const totalCost = pos.shares * pos.avgCost;
    const gainLoss = marketValue - totalCost;
    const gainLossPct = totalCost > 0 ? (gainLoss / totalCost) * 100 : 0;
    const dayChange = pos.prevClose > 0
      ? pos.shares * (pos.currentPrice - pos.prevClose) : 0;

    return {
      symbol, shares: pos.shares, avgCost: pos.avgCost,
      currentPrice: pos.currentPrice,
      marketValue: parseFloat(marketValue.toFixed(2)),
      totalCost: parseFloat(totalCost.toFixed(2)),
      gainLoss: parseFloat(gainLoss.toFixed(2)),
      gainLossPct: parseFloat(gainLossPct.toFixed(2)),
      dayChange: parseFloat(dayChange.toFixed(2)),
      lastUpdated: pos.lastUpdated
    };
  }

  getPortfolioSummary() {
    let totalMarketValue = 0, totalCost = 0, totalDayChange = 0;
    const positions = [];

    for (const symbol of this.positions.keys()) {
      const summary = this.getPositionSummary(symbol);
      if (summary) {
        positions.push(summary);
        totalMarketValue += summary.marketValue;
        totalCost += summary.totalCost;
        totalDayChange += summary.dayChange;
      }
    }

    return {
      positions,
      totalMarketValue: parseFloat(totalMarketValue.toFixed(2)),
      totalCost: parseFloat(totalCost.toFixed(2)),
      totalGainLoss: parseFloat((totalMarketValue - totalCost).toFixed(2)),
      totalGainLossPct: parseFloat(
        (totalCost > 0 ? ((totalMarketValue - totalCost) / totalCost) * 100 : 0).toFixed(2)
      ),
      totalDayChange: parseFloat(totalDayChange.toFixed(2))
    };
  }
}
```

## Price Alert System

### Alert Types

| Alert Type | Condition | Description |
|------------|-----------|-------------|
| Price Above | `price >= target` | Notify when price rises to or above target |
| Price Below | `price <= target` | Notify when price falls to or below target |
| Percent Change Up | `changePct >= threshold` | Notify on day gain percentage threshold |
| Percent Change Down | `changePct <= -threshold` | Notify on day loss percentage threshold |
| Volume Spike | `volume >= threshold` | Notify on unusual volume |

### Setting Alerts (Client-Side)

```javascript
class AlertManager {
  constructor(pubnub, userId) {
    this.pubnub = pubnub;
    this.userId = userId;
    this.alertChannel = `alerts.${userId}`;
  }

  async createAlert(alert) {
    const alertId = `${alert.symbol}_${alert.type}_${Date.now()}`;
    await this.pubnub.publish({
      channel: 'system.alerts.register',
      message: {
        id: alertId, userId: this.userId,
        symbol: alert.symbol, type: alert.type,
        target: alert.target, direction: alert.direction || null,
        createdAt: Date.now(), triggered: false
      }
    });
    return alertId;
  }

  async deleteAlert(alertId) {
    await this.pubnub.publish({
      channel: 'system.alerts.delete',
      message: { id: alertId, userId: this.userId }
    });
  }

  listen(onAlert) {
    this.pubnub.addListener({
      message: (event) => {
        if (event.channel === this.alertChannel) onAlert(event.message);
      }
    });
    this.pubnub.subscribe({ channels: [this.alertChannel] });
  }
}
```

### PubNub Function for Alert Evaluation

```javascript
// PubNub Function: After Publish on quotes.* channels
export default (request) => {
  const kvstore = require('kvstore');
  const quote = request.message;

  return kvstore.get(`alerts_${quote.symbol}`).then((data) => {
    if (!data) return request.ok();
    const alerts = JSON.parse(data);
    const triggered = [];

    alerts.forEach((alert) => {
      if (alert.triggered) return;
      let shouldFire = false;

      if (alert.type === 'price_above') shouldFire = quote.price >= alert.target;
      if (alert.type === 'price_below') shouldFire = quote.price <= alert.target;
      if (alert.type === 'pct_change_up') shouldFire = quote.changePct >= alert.target;
      if (alert.type === 'pct_change_down') shouldFire = quote.changePct <= -alert.target;
      if (alert.type === 'volume_spike') shouldFire = quote.volume >= alert.target;

      if (shouldFire) {
        alert.triggered = true;
        triggered.push(alert);
        pubnub.fire({
          channel: `alerts.${alert.userId}`,
          message: {
            alertId: alert.id, symbol: quote.symbol,
            type: alert.type, target: alert.target,
            actual: quote.price, triggeredAt: Date.now()
          }
        });
      }
    });

    if (triggered.length > 0) {
      return kvstore.set(`alerts_${quote.symbol}`, JSON.stringify(alerts))
        .then(() => request.ok());
    }
    return request.ok();
  });
};
```

## Gain/Loss Tracking

### Calculation and Formatting

```javascript
function calculateGainLoss(position, quote) {
  const { shares, avgCost } = position;
  const totalGL = shares * (quote.price - avgCost);
  const totalGLPct = ((quote.price - avgCost) / avgCost) * 100;
  const dayGL = shares * (quote.price - quote.prevClose);
  const dayGLPct = ((quote.price - quote.prevClose) / quote.prevClose) * 100;

  return {
    totalGainLoss: parseFloat(totalGL.toFixed(2)),
    totalGainLossPct: parseFloat(totalGLPct.toFixed(2)),
    dayGainLoss: parseFloat(dayGL.toFixed(2)),
    dayGainLossPct: parseFloat(dayGLPct.toFixed(2))
  };
}

```

## Historical Data and Charting Support

### Fetching Quote History from PubNub

```javascript
async function fetchQuoteHistory(pubnub, symbol, minutes = 60) {
  const startTime = (Date.now() - minutes * 60 * 1000) * 10000;
  const result = await pubnub.fetchMessages({
    channels: [`quotes.${symbol}`],
    start: startTime.toString(),
    count: 100
  });
  const messages = result.channels[`quotes.${symbol}`] || [];
  return messages.map((m) => ({
    price: m.message.price, volume: m.message.volume,
    high: m.message.high, low: m.message.low,
    timestamp: m.message.timestamp
  }));
}
```

### Building OHLCV Bars from Tick Data

```javascript
function aggregateToOHLCV(ticks, intervalMs = 60000) {
  const bars = new Map();
  ticks.forEach((tick) => {
    const barTime = Math.floor(tick.timestamp / intervalMs) * intervalMs;
    if (!bars.has(barTime)) {
      bars.set(barTime, {
        open: tick.price, high: tick.price,
        low: tick.price, close: tick.price,
        volume: tick.volume || 0, timestamp: barTime
      });
    } else {
      const bar = bars.get(barTime);
      bar.high = Math.max(bar.high, tick.price);
      bar.low = Math.min(bar.low, tick.price);
      bar.close = tick.price;
      bar.volume += tick.volume || 0;
    }
  });
  return Array.from(bars.values()).sort((a, b) => a.timestamp - b.timestamp);
}
```

## Stale Data Detection

```javascript
const STALE_THRESHOLD_MS = 30000;

function isQuoteStale(quote) {
  if (!quote || !quote.timestamp) return true;
  return (Date.now() - quote.timestamp) > STALE_THRESHOLD_MS;
}

async function recoverAfterReconnect(pubnub, watchlist, portfolio) {
  const symbols = await watchlist.listSymbols();
  const lastQuotes = await fetchLastQuotes(pubnub, symbols);
  for (const [symbol, quote] of Object.entries(lastQuotes)) {
    portfolio.updatePrice(symbol, quote);
  }
  return portfolio.getPortfolioSummary();
}
```

## Best Practices

- **Persist watchlist state** in both your application database and PubNub channel groups. Sync on startup to handle cases where one side drifted.
- **Fetch last known quotes** via `fetchMessages` with `count: 1` on app load so users see current prices immediately before live updates arrive.
- **Use channel groups** instead of subscribing to individual channels. This simplifies subscription management and keeps the subscribe call clean regardless of watchlist size.
- **Implement stale data indicators** on the client. If a quote has not updated within 30 seconds during market hours, display a visual indicator so users know the data may be outdated.
- **Fire alerts with `pubnub.fire()`** instead of `publish()` to avoid storing alert notifications in history. Alerts are ephemeral; if the user is not connected, deliver via push notification instead.
- **Throttle portfolio recalculation** on the client side. Recalculating total portfolio value on every tick can be expensive. Debounce to every 250-500ms.
- **Store triggered alert state** in the KV store to prevent duplicate alert notifications. Reset triggered flags at the start of each trading day.
- **Handle partial data gracefully** in portfolio calculations. If a quote has not arrived for a symbol, show the last known price with a timestamp rather than showing zero.
- **Clean up channel groups** when users delete their account or watchlist. Orphaned channel groups continue to consume subscription resources.