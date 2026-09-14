# PubNub Portfolio Tracking

Watchlist channel groups, quote subscribe, alert Functions, history catch-up, and stale detection. Gain/loss and OHLCV math are out of scope.

## Watchlists with channel groups

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

### App load: sync group + last quotes

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

Subscribe clients to `quotes.*` (or the watchlist group). Portfolio value math stays in the app, not this skill.

## Price alerts (Function + KV)

Register on `system.alerts.register`; notify on `alerts.{userId}`.

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
        target: alert.target, createdAt: Date.now(), triggered: false
      }
    });
    return alertId;
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

After Publish on `quotes.*` (link [functions-patterns](../../pubnub-functions/references/functions-patterns.md) for handler shape):

```javascript
export default async (request) => {
  const kvstore = require('kvstore');
  const pubnub = require('pubnub');
  const quote = request.message;
  const data = await kvstore.get(`alerts_${quote.symbol}`);
  if (!data) return request.ok();
  const alerts = JSON.parse(data);
  let dirty = false;
  for (const alert of alerts) {
    if (alert.triggered) continue;
    const fire = (alert.type === 'price_above' && quote.price >= alert.target)
      || (alert.type === 'price_below' && quote.price <= alert.target);
    if (!fire) continue;
    alert.triggered = true;
    dirty = true;
    pubnub.fire({
      channel: `alerts.${alert.userId}`,
      message: { alertId: alert.id, symbol: quote.symbol, type: alert.type, actual: quote.price, triggeredAt: Date.now() }
    });
  }
  if (dirty) await kvstore.set(`alerts_${quote.symbol}`, JSON.stringify(alerts));
  return request.ok();
};
```

Use `fire` so alert notifications are not stored in history. Reset triggered flags on the operator's trading-day boundary.

## History catch-up

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
    timestamp: m.message.timestamp
  }));
}
```

Do not aggregate OHLCV bars in this skill.

## Stale data and reconnect

```javascript
const STALE_THRESHOLD_MS = 30000;

function isQuoteStale(quote) {
  if (!quote || !quote.timestamp) return true;
  return (Date.now() - quote.timestamp) > STALE_THRESHOLD_MS;
}

async function recoverAfterReconnect(pubnub, watchlist) {
  const symbols = await watchlist.listSymbols();
  return fetchLastQuotes(pubnub, symbols);
}
```

## Best Practices

- **Persist watchlist** in your DB and PubNub channel groups; sync on startup.
- **`fetchMessages` with `count: 1`** on load and after reconnect.
- **Channel groups** instead of per-symbol subscribe.
- **Stale indicators** when ticks stop during an expected-live window.
- **`pubnub.fire()` for alerts**; push if the user is offline.
- **Clean up channel groups** when a watchlist or account is deleted.
