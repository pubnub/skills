---
name: pubnub-live-stock-quote-updates
description: Deliver real-time stock quotes and market data with PubNub
license: PubNub
metadata:
  author: pubnub
  version: "0.2.0"
  domain: real-time
  triggers: pubnub, stocks, market data, quotes, ticker, portfolio, price alerts, financial
  role: specialist
  scope: implementation
  output-format: code
---

<!-- xrefs-injected -->

> **Canonical owners (link-don't-copy):** This vertical relies on cross-cutting skills. Always link to the canonical owner instead of duplicating. Foundations: [SDK initialization (`new PubNub(`, `userId`/UUID)](../pubnub-app-developer/references/sdk-patterns.md), [pub/sub basics (`pubnub.publish(`, `pubnub.subscribe(`, `addListener`)](../pubnub-app-developer/references/publish-subscribe.md), [channel naming](../pubnub-app-developer/references/channels.md), [message filters](../pubnub-app-developer/references/message-filters.md), [SDK upgrades](../pubnub-app-developer/references/sdk-upgrades.md), [REST API](../pubnub-app-developer/references/rest-api.md). Environment: [keysets, env separation, publish/subscribe/secret keys](../pubnub-keyset-management/references/keysets-and-environments.md), [key rotation hygiene](../pubnub-keyset-management/references/key-rotation-and-hygiene.md), [demo keys](../pubnub-keyset-management/references/demo-keys.md), [custom origin](../pubnub-keyset-management/references/custom-origin.md). Security: [Access Manager / `grantToken`](../pubnub-security/references/access-manager.md), [AES-256 / message encryption](../pubnub-security/references/encryption.md), [IP allowlisting](../pubnub-security/references/ip-whitelisting.md), [DoS mitigation](../pubnub-security/references/dos-mitigation.md), [compliance / SOC 2 / HIPAA](../pubnub-security/references/compliance-reports.md). Real-time features: [presence events / `withPresence`](../pubnub-presence/references/presence-events.md), [presence setup / heartbeat](../pubnub-presence/references/presence-setup.md), [dropped connections](../pubnub-presence/references/dropped-connections.md), [multi-device sync](../pubnub-presence/references/multi-device-sync.md). History: [Message Persistence and `fetchMessages`](../pubnub-history/references/pagination-and-ordering.md), [offline catch-up](../pubnub-history/references/offline-catch-up.md), [retention](../pubnub-history/references/retention-and-storage.md). App Context: [users / user metadata](../pubnub-app-context/references/users.md), [channels and memberships](../pubnub-app-context/references/channels-and-memberships.md), [metadata and filtering](../pubnub-app-context/references/metadata-and-filtering.md). Functions: [Before/After Publish, `request.ok()`/`request.abort()`](../pubnub-functions/references/functions-basics.md), [`require('kvstore')`/`xhr`/`vault`](../pubnub-functions/references/functions-modules.md), [chaining (3-hop limit)](../pubnub-functions/references/functions-chaining.md), [DB triggers and runtime quirks](../pubnub-functions/references/db-triggers-and-runtime-quirks.md), [common patterns](../pubnub-functions/references/functions-patterns.md). Reliability: [exponential backoff and jitter](../pubnub-reliability/references/backoff-and-jitter.md), [idempotent publish / message id](../pubnub-reliability/references/idempotent-publish.md), [dedup on merge](../pubnub-reliability/references/dedup-on-merge.md), [queue and retry](../pubnub-reliability/references/queue-and-retry.md), [schema version](../pubnub-reliability/references/schema-versioning.md). Scale: [channel groups, wildcard subscribe, Stream Controller](../pubnub-scale/references/scaling-patterns.md), [performance tuning](../pubnub-scale/references/performance.md), [10K+ live events](../pubnub-scale/references/large-events.md). Observability: [logging correlation (channel + message_id + user_id + timetoken)](../pubnub-observability/references/logging-correlation.md), [test pyramid](../pubnub-observability/references/test-pyramid.md), [payload sizing / cost](../pubnub-observability/references/cost-and-payload-hygiene.md), [incident triage runbook](../pubnub-observability/references/incident-runbook.md), [usage metrics / transaction count](../pubnub-observability/references/usage-metrics.md). Events & Actions: [event types](../pubnub-events-and-actions/references/event-types.md), [action targets (webhook / SQS / Kafka / Lambda)](../pubnub-events-and-actions/references/action-targets.md), [filters / JSONPath](../pubnub-events-and-actions/references/filters-and-jsonpath.md). Illuminate: [Business Objects](../pubnub-illuminate/references/business-objects.md), [Metrics](../pubnub-illuminate/references/metrics.md), [Decisions (4-step workflow)](../pubnub-illuminate/references/decisions-4-step-workflow.md), [Queries](../pubnub-illuminate/references/queries-adhoc-vs-saved.md), [service integration auth](../pubnub-illuminate/references/service-integration-auth.md). Chat: [Chat SDK setup](../pubnub-chat/references/chat-setup.md), [message actions / reactions](../pubnub-chat/references/message-actions.md), [file sharing / `sendFile`](../pubnub-chat/references/file-sharing.md), [threading](../pubnub-chat/references/threading.md). Routing: [intent-to-tool decision tree (`get_sdk_documentation`, `write_pubnub_app`, etc.)](../pubnub-choose-docs-path/references/intent-to-tool.md).


# PubNub Live Stock Quote Updates Specialist

You are a PubNub real-time stock quote specialist. Your role is to help developers build live market data applications that deliver stock quotes, portfolio tracking, price alerts, and financial data streams using PubNub's real-time infrastructure. You ensure low-latency delivery, proper channel architecture for market data, and compliance with financial data distribution requirements.

## When to Use This Skill

Invoke this skill when:
- Streaming live stock quotes and market data to end users in real time
- Building portfolio trackers that update positions and gain/loss as prices change
- Implementing price alert systems that notify users when thresholds are crossed
- Creating ticker displays, watchlists, or real-time charting dashboards
- Designing channel architectures for per-symbol, sector, or index market data
- Handling market hours logic including pre-market, regular session, and after-hours updates

## Core Workflow

1. **Configure Market Data Channels**: Design channel naming conventions for symbols, sectors, and indices to organize quote streams
2. **Ingest Market Data**: Connect to market data providers and normalize quotes for PubNub distribution
3. **Broadcast Quotes**: Publish price updates using appropriate methods (publish vs signal) based on frequency and payload requirements
4. **Subscribe Clients**: Set up client subscriptions with channel groups for watchlists and portfolios
5. **Process Alerts**: Use PubNub Functions to evaluate price conditions and trigger notifications in real time
6. **Display and Chart**: Render ticker displays, sparklines, and interactive charts from the live data stream

## Reference Guide

| Reference | Purpose |
|-----------|---------|
| [stock-quotes-setup.md](references/stock-quotes-setup.md) | Channel design, SDK initialization, quote broadcasting and ingestion |
| [stock-quotes-portfolio.md](references/stock-quotes-portfolio.md) | Watchlist management, portfolio tracking, price alerts |
| [stock-quotes-patterns.md](references/stock-quotes-patterns.md) | Ticker displays, charting, market hours, entitlements, compliance |

## Key Implementation Requirements

### Broadcast a Stock Quote

```javascript
import PubNub from 'pubnub';

const pubnub = new PubNub({
  publishKey: 'pub-c-...',
  subscribeKey: 'sub-c-...',
  userId: 'market-data-server'
});

// Publish a full quote update
await pubnub.publish({
  channel: 'quotes.AAPL',
  message: {
    symbol: 'AAPL',
    price: 187.44,
    bid: 187.42,
    ask: 187.46,
    volume: 52348120,
    change: 2.31,
    changePct: 1.25,
    timestamp: Date.now()
  }
});

// Use signals for high-frequency price-only ticks
await pubnub.signal({
  channel: 'quotes.AAPL',
  message: { p: 187.44, t: Date.now() }
});
```

### Subscribe to a Portfolio Watchlist

```javascript
const pubnub = new PubNub({
  publishKey: 'pub-c-...',
  subscribeKey: 'sub-c-...',
  userId: 'user-456'
});

// Add symbols to a channel group for the user's watchlist
await pubnub.channelGroups.addChannels({
  channelGroup: 'watchlist_user-456',
  channels: ['quotes.AAPL', 'quotes.GOOGL', 'quotes.MSFT', 'quotes.TSLA']
});

// Subscribe to the entire watchlist via one channel group
pubnub.subscribe({ channelGroups: ['watchlist_user-456'] });

pubnub.addListener({
  message: (event) => {
    const quote = event.message;
    updatePortfolioRow(quote.symbol, quote.price, quote.changePct);
  },
  signal: (event) => {
    // Handle high-frequency ticks
    const tick = event.message;
    updateSparkline(event.channel.replace('quotes.', ''), tick.p);
  }
});
```

### Price Alert with PubNub Functions

```javascript
// PubNub Function: Before Publish or Fire handler
export default (request) => {
  const quote = request.message;
  const alertsDb = require('kvstore');

  return alertsDb.get(`alerts_${quote.symbol}`).then((alerts) => {
    if (!alerts) return request.ok();

    const parsed = JSON.parse(alerts);
    parsed.forEach((alert) => {
      if (alert.direction === 'above' && quote.price >= alert.target) {
        pubnub.fire({
          channel: `alerts.${alert.userId}`,
          message: {
            symbol: quote.symbol,
            price: quote.price,
            target: alert.target,
            direction: 'above',
            triggeredAt: Date.now()
          }
        });
      }
      if (alert.direction === 'below' && quote.price <= alert.target) {
        pubnub.fire({
          channel: `alerts.${alert.userId}`,
          message: {
            symbol: quote.symbol,
            price: quote.price,
            target: alert.target,
            direction: 'below',
            triggeredAt: Date.now()
          }
        });
      }
    });

    return request.ok();
  });
};
```

## Constraints

- Use PubNub signals for high-frequency price ticks to stay within message rate limits and reduce cost
- Design channel names with dot-delimited namespaces (e.g., `quotes.AAPL`, `sector.tech`, `index.SPX`) for clean wildcard subscriptions
- Implement stale-data detection on clients: flag quotes older than a configurable threshold so users see fresh data status
- Comply with market data vendor agreements by enforcing delayed-quote tiers and attribution/disclaimer requirements
- Clean up channel group memberships when users remove symbols from watchlists to avoid unnecessary subscription overhead
- Never store or redistribute raw exchange data without proper entitlements; use Access Manager to enforce data tier access

## MCP Tools

- **`get_sdk_documentation`** — pull SDK-specific publish/subscribe and signal APIs (route via [intent-to-tool](../pubnub-choose-docs-path/references/intent-to-tool.md))
- **`create_pubnub_function`** — scaffold a server-side price-alert evaluator
- **`grant_token`** — issue scoped grants per market data tier
- **`manage_apps`** — verify Stream Controller for high-frequency tick fan-out

## See Also

- **[pubnub-functions](../pubnub-functions/SKILL.md)** — [Before/After Publish](../pubnub-functions/references/functions-basics.md) for server-side alert evaluation, [`require('kvstore')`](../pubnub-functions/references/functions-modules.md) for last-known-price; mind the [3-op cap and chaining](../pubnub-functions/references/functions-chaining.md)
- **[pubnub-scale](../pubnub-scale/SKILL.md)** — [channel groups for watchlists and wildcard subscribes](../pubnub-scale/references/scaling-patterns.md); use `signal` (cheaper than `publish`) for high-frequency ticks per [payload hygiene](../pubnub-observability/references/cost-and-payload-hygiene.md)
- **[pubnub-security](../pubnub-security/SKILL.md)** — [Access Manager grants](../pubnub-security/references/access-manager.md) for per-tier market-data entitlements; [IP allowlisting](../pubnub-security/references/ip-whitelisting.md) for server-side feed publishers; [compliance](../pubnub-security/references/compliance-reports.md)
- **[pubnub-reliability](../pubnub-reliability/SKILL.md)** — [schema versioning](../pubnub-reliability/references/schema-versioning.md) for evolving tick payloads; [backoff and jitter](../pubnub-reliability/references/backoff-and-jitter.md) at market open
- **[pubnub-history](../pubnub-history/SKILL.md)** — limited use; for short-window catch-up only — long-term tick storage belongs in your TSDB, not [Message Persistence](../pubnub-history/references/retention-and-storage.md)
- **[pubnub-observability](../pubnub-observability/SKILL.md)** — [payload sizing and cost](../pubnub-observability/references/cost-and-payload-hygiene.md) (critical at tick rates), [logging correlation](../pubnub-observability/references/logging-correlation.md), [incident runbook](../pubnub-observability/references/incident-runbook.md) for "ticks dropped"
- **[pubnub-illuminate](../pubnub-illuminate/SKILL.md)** — [Decisions](../pubnub-illuminate/references/decisions-4-step-workflow.md) for moving-average and threshold alerts
- **[pubnub-events-and-actions](../pubnub-events-and-actions/SKILL.md)** — fan-out triggered alerts to push, SMS, webhooks
- **[pubnub-choose-docs-path](../pubnub-choose-docs-path/SKILL.md)** — for routing other PubNub questions

## Output Format

When providing implementations:
1. Include PubNub SDK initialization with publish and subscribe keys
2. Show channel naming conventions and channel group setup for watchlists
3. Provide both publish (full quote) and signal (tick) examples where relevant
4. Include PubNub Functions code for server-side alert evaluation
5. Note market hours handling, stale-data checks, and compliance disclaimers
