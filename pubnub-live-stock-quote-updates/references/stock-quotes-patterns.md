# PubNub Stock Quote Patterns

This reference covers ticker display implementations, real-time charting integration, market hours handling, data entitlements and access control, multi-exchange support, and regulatory compliance for stock quote applications built with PubNub.

## Ticker Display Implementation

### Scrolling Ticker

A scrolling ticker continuously displays symbols and prices moving across the screen.

```javascript
class ScrollingTicker {
  constructor(pubnub, containerEl, symbols) {
    this.pubnub = pubnub;
    this.container = containerEl;
    this.quotes = new Map();
    this.symbols = symbols;
    this.initSubscription();
    this.startAnimation();
  }

  initSubscription() {
    this.pubnub.addListener({
      message: (event) => {
        const q = event.message;
        this.quotes.set(q.symbol, {
          price: q.price, change: q.change, changePct: q.changePct
        });
      },
      signal: (event) => {
        const symbol = event.channel.replace('quotes.', '');
        const existing = this.quotes.get(symbol) || {};
        this.quotes.set(symbol, { ...existing, price: event.message.p });
      }
    });
    this.pubnub.subscribe({
      channels: this.symbols.map((s) => `quotes.${s}`)
    });
  }

  renderTickerItem(symbol) {
    const q = this.quotes.get(symbol);
    if (!q) return `${symbol}: --`;
    const sign = q.change >= 0 ? '+' : '';
    return `${symbol} $${q.price.toFixed(2)} ${sign}${q.changePct.toFixed(2)}%`;
  }

  startAnimation() {
    const render = () => {
      this.container.textContent = this.symbols
        .map((s) => this.renderTickerItem(s)).join('    |    ');
      requestAnimationFrame(render);
    };
    render();
  }

  destroy() { this.pubnub.unsubscribeAll(); }
}
```

### Sparkline Display

Sparklines show mini price charts inline with the quote.

```javascript
class SparklineTracker {
  constructor(maxPoints = 50) {
    this.data = new Map();
    this.maxPoints = maxPoints;
  }

  addPoint(symbol, price, timestamp) {
    if (!this.data.has(symbol)) this.data.set(symbol, []);
    const points = this.data.get(symbol);
    points.push({ price, timestamp });
    if (points.length > this.maxPoints) points.shift();
  }

  renderSVG(symbol, width = 100, height = 30) {
    const points = this.data.get(symbol) || [];
    if (points.length < 2) return '';
    const prices = points.map((p) => p.price);
    const min = Math.min(...prices);
    const range = (Math.max(...prices) - min) || 1;

    const coords = points.map((p, i) => {
      const x = (i / (points.length - 1)) * width;
      const y = height - ((p.price - min) / range) * height;
      return `${x},${y}`;
    });
    const color = prices[prices.length - 1] >= prices[0] ? '#22c55e' : '#ef4444';
    return `<svg width="${width}" height="${height}"><polyline points="${coords.join(' ')}" fill="none" stroke="${color}" stroke-width="1.5"/></svg>`;
  }
}
```

### Display Format Comparison

| Format | Best For | Update Rate | Complexity |
|--------|----------|-------------|------------|
| Scrolling Ticker | Broadcast displays, TV overlays | Every tick | Low |
| Grid / Table | Watchlists, dashboards | 250ms-1s debounced | Medium |
| Sparkline | Inline trend visualization | Every tick, batched render | Medium |
| Candlestick Chart | Technical analysis | 1-minute bars | High |
| Heat Map | Sector/market overview | 5-15 seconds | Medium |

## Real-Time Charting Integration

### Connecting PubNub to a Charting Library

```javascript
class RealTimeChart {
  constructor(pubnub, symbol, chartInstance) {
    this.pubnub = pubnub;
    this.symbol = symbol;
    this.chart = chartInstance;
    this.currentBar = null;
    this.barIntervalMs = 60000; // 1-minute bars
  }

  start() {
    this.pubnub.addListener({
      message: (event) => this.updateBar(event.message.price, event.message.volume, event.message.timestamp),
      signal: (event) => this.updateBar(event.message.p, 0, event.message.t)
    });
    this.pubnub.subscribe({ channels: [`quotes.${this.symbol}`] });
  }

  updateBar(price, volume, timestamp) {
    const barTime = Math.floor(timestamp / this.barIntervalMs) * this.barIntervalMs;
    if (!this.currentBar || this.currentBar.time !== barTime) {
      if (this.currentBar) this.chart.addCompletedBar(this.currentBar);
      this.currentBar = {
        time: barTime, open: price, high: price,
        low: price, close: price, volume: volume
      };
    } else {
      this.currentBar.high = Math.max(this.currentBar.high, price);
      this.currentBar.low = Math.min(this.currentBar.low, price);
      this.currentBar.close = price;
      this.currentBar.volume += volume;
    }
    this.chart.updateCurrentBar(this.currentBar);
  }

  stop() { this.pubnub.unsubscribe({ channels: [`quotes.${this.symbol}`] }); }
}
```

## Market Hours Handling

### Market Session Definitions

| Session | Time (ET) | Channel Suffix | Notes |
|---------|-----------|---------------|-------|
| Pre-Market | 4:00 AM - 9:30 AM | `.pre` | Lower liquidity, wider spreads |
| Regular | 9:30 AM - 4:00 PM | (none) | Primary session |
| After-Hours | 4:00 PM - 8:00 PM | `.post` | Lower liquidity, wider spreads |
| Weekend/Holiday | N/A | N/A | No updates expected |

### Market Hours Utility

```javascript
class MarketHours {
  constructor(timezone = 'America/New_York') {
    this.timezone = timezone;
    this.holidays = new Set([
      '2025-01-01', '2025-01-20', '2025-02-17', '2025-04-18',
      '2025-05-26', '2025-06-19', '2025-07-04', '2025-09-01',
      '2025-11-27', '2025-12-25'
    ]);
  }

  getCurrentSession() {
    const now = new Date();
    const et = new Date(now.toLocaleString('en-US', { timeZone: this.timezone }));
    const timeDecimal = et.getHours() + et.getMinutes() / 60;
    const day = et.getDay();
    const dateStr = et.toISOString().split('T')[0];

    if (day === 0 || day === 6 || this.holidays.has(dateStr)) {
      return { session: 'closed', label: 'Market Closed', isTrading: false };
    }
    if (timeDecimal >= 4.0 && timeDecimal < 9.5) {
      return { session: 'pre-market', label: 'Pre-Market', isTrading: true };
    }
    if (timeDecimal >= 9.5 && timeDecimal < 16.0) {
      return { session: 'regular', label: 'Regular Hours', isTrading: true };
    }
    if (timeDecimal >= 16.0 && timeDecimal < 20.0) {
      return { session: 'after-hours', label: 'After Hours', isTrading: true };
    }
    return { session: 'closed', label: 'Market Closed', isTrading: false };
  }

  isStaleExpected() {
    return !this.getCurrentSession().isTrading;
  }
}
```

## Data Entitlements and Access Control

### Tier Definitions

| Tier | Data | Delay | Channel Prefix |
|------|------|-------|----------------|
| Free | Basic quotes | 15-minute delay | `delayed.` |
| Standard | Real-time quotes | None | `quotes.` |
| Premium | Real-time + Level 2 | None | `premium.` |
| Professional | Full depth, trades | None | `pro.` |

### Access Manager Configuration

```javascript
// Server-side: Grant access based on user tier
async function grantAccess(pubnub, userId, tier) {
  const channelPatterns = {
    free: ['delayed.*'],
    standard: ['delayed.*', 'quotes.*', 'index.*'],
    premium: ['delayed.*', 'quotes.*', 'index.*', 'premium.*'],
    professional: ['delayed.*', 'quotes.*', 'index.*', 'premium.*', 'pro.*']
  };
  const channels = channelPatterns[tier] || channelPatterns.free;

  const token = await pubnub.grantToken({
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
  return token;
}

function getChannelForTier(symbol, tier) {
  if (tier === 'free') return `delayed.${symbol}`;
  if (tier === 'professional') return `pro.${symbol}`;
  return `quotes.${symbol}`;
}
```

## Multi-Exchange Support

### Exchange Configuration

| Exchange | Timezone | Regular Hours (Local) | Channel Convention |
|----------|----------|----------------------|-------------------|
| NYSE / NASDAQ | America/New_York | 9:30 AM - 4:00 PM | `quotes.<SYMBOL>` |
| LSE | Europe/London | 8:00 AM - 4:30 PM | `quotes.LON.<SYMBOL>` |
| TSE | Asia/Tokyo | 9:00 AM - 3:00 PM | `quotes.TYO.<SYMBOL>` |
| HKEX | Asia/Hong_Kong | 9:30 AM - 4:00 PM | `quotes.HKG.<SYMBOL>` |

### Multi-Exchange Subscription

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

## Regulatory Compliance

### Delayed Quote Disclaimers

```javascript
function getDisclaimer(tier, exchange) {
  const disclaimers = {
    free: {
      NYSE: 'Quotes delayed by at least 15 minutes. Data provided by NYSE.',
      NASDAQ: 'Quotes delayed by at least 15 minutes. Data provided by NASDAQ.',
      default: 'Quotes may be delayed by 15-20 minutes. Not for trading purposes.'
    },
    standard: {
      default: 'Real-time quotes for informational purposes only. Not investment advice.'
    }
  };
  const tierDisclaimers = disclaimers[tier] || disclaimers.free;
  return tierDisclaimers[exchange] || tierDisclaimers.default;
}

function renderAttribution(dataSource, tier) {
  const isDelayed = tier === 'free';
  return `<div class="data-attribution">
    <span>Market data by ${dataSource}</span>
    ${isDelayed ? '<span class="delay-badge">15 MIN DELAY</span>' : ''}
    <span>As of ${new Date().toLocaleTimeString()}</span>
  </div>`;
}
```

## Mobile Ticker (Kotlin / Android)

```kotlin
import com.pubnub.api.PubNub
import com.pubnub.api.callbacks.SubscribeCallback
import com.pubnub.api.models.consumer.pubsub.PNMessageResult

class StockTickerViewModel(private val pubnub: PubNub) : ViewModel() {
    private val _quotes = MutableLiveData<Map<String, QuoteData>>()
    val quotes: LiveData<Map<String, QuoteData>> = _quotes

    fun subscribeToSymbols(symbols: List<String>) {
        val channels = symbols.map { "quotes.$it" }
        pubnub.addListener(object : SubscribeCallback() {
            override fun message(pubnub: PubNub, message: PNMessageResult) {
                val json = message.message.asJsonObject
                val symbol = json.get("symbol").asString
                val current = _quotes.value?.toMutableMap() ?: mutableMapOf()
                current[symbol] = QuoteData(
                    symbol, json.get("price").asDouble,
                    json.get("change").asDouble, json.get("changePct").asDouble
                )
                _quotes.postValue(current)
            }
        })
        pubnub.subscribe(channels = channels)
    }
}

data class QuoteData(
    val symbol: String, val price: Double,
    val change: Double, val changePct: Double
)
```

## Error Handling Patterns

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

function validateQuote(quote) {
  const errors = [];
  if (!quote.symbol || typeof quote.symbol !== 'string') errors.push('Invalid symbol');
  if (typeof quote.price !== 'number' || quote.price <= 0) errors.push('Invalid price');
  if (quote.timestamp && (Date.now() - quote.timestamp) > 300000) errors.push('Stale timestamp');
  if (quote.bid && quote.ask && quote.bid > quote.ask) errors.push('Crossed market');
  return { valid: errors.length === 0, errors, quote };
}
```

## Best Practices

- **Debounce UI rendering** when receiving high-frequency updates. Use `requestAnimationFrame` or a 100-250ms debounce to avoid excessive DOM updates that degrade performance.
- **Show market session status** prominently in the UI. Users should always know whether they are viewing pre-market, regular, after-hours, or closed-market data.
- **Implement color flash on price change**: briefly highlight the price cell green on uptick and red on downtick, then fade back to neutral for instant visual feedback.
- **Cache last known quotes locally** (e.g., in localStorage) so the app can show recent data instantly on reload, even before PubNub connects.
- **Enforce data tier access** with PubNub Access Manager. Never rely on client-side checks alone to restrict premium data.
- **Display required disclaimers** based on the data tier and exchange. Free-tier users must see the delayed-data notice with data source attribution.
- **Handle market holidays and half-days** in your market hours logic. Maintain an updated holiday calendar to avoid false stale-data warnings.
- **Use separate channels for extended-hours data** rather than mixing pre/after-hours quotes into the regular channel. This lets users opt in or out.
- **Validate all incoming quote data** before rendering. Drop quotes with invalid prices, crossed bid/ask, or timestamps too far in the past.
- **Test with realistic data volumes**. Popular symbols can generate hundreds of ticks per second during volatile periods. Ensure your rendering pipeline handles peak load.
