---
name: pubnub-live-betting-casino
description: Build real-time betting and casino game platforms with PubNub
license: PubNub
metadata:
  author: pubnub
  version: "0.2.0"
  domain: real-time
  triggers: pubnub, betting, casino, odds, wager, live betting, in-play, game state
  role: specialist
  scope: implementation
  output-format: code
---

<!-- xrefs-injected -->

> **Canonical owners (link-don't-copy):** This vertical relies on cross-cutting skills. Always link to the canonical owner instead of duplicating. Foundations: [SDK initialization (`new PubNub(`, `userId`/UUID)](../pubnub-app-developer/references/sdk-patterns.md), [pub/sub basics (`pubnub.publish(`, `pubnub.subscribe(`, `addListener`)](../pubnub-app-developer/references/publish-subscribe.md), [channel naming](../pubnub-app-developer/references/channels.md), [message filters](../pubnub-app-developer/references/message-filters.md), [SDK upgrades](../pubnub-app-developer/references/sdk-upgrades.md), [REST API](../pubnub-app-developer/references/rest-api.md). Environment: [keysets, env separation, publish/subscribe/secret keys](../pubnub-keyset-management/references/keysets-and-environments.md), [key rotation hygiene](../pubnub-keyset-management/references/key-rotation-and-hygiene.md), [demo keys](../pubnub-keyset-management/references/demo-keys.md), [custom origin](../pubnub-keyset-management/references/custom-origin.md). Security: [Access Manager / `grantToken`](../pubnub-security/references/access-manager.md), [AES-256 / message encryption](../pubnub-security/references/encryption.md), [IP allowlisting](../pubnub-security/references/ip-whitelisting.md), [DoS mitigation](../pubnub-security/references/dos-mitigation.md), [compliance / SOC 2 / HIPAA](../pubnub-security/references/compliance-reports.md). Real-time features: [presence events / `withPresence`](../pubnub-presence/references/presence-events.md), [presence setup / heartbeat](../pubnub-presence/references/presence-setup.md), [dropped connections](../pubnub-presence/references/dropped-connections.md), [multi-device sync](../pubnub-presence/references/multi-device-sync.md). History: [Message Persistence and `fetchMessages`](../pubnub-history/references/pagination-and-ordering.md), [offline catch-up](../pubnub-history/references/offline-catch-up.md), [retention](../pubnub-history/references/retention-and-storage.md). App Context: [users / user metadata](../pubnub-app-context/references/users.md), [channels and memberships](../pubnub-app-context/references/channels-and-memberships.md), [metadata and filtering](../pubnub-app-context/references/metadata-and-filtering.md). Functions: [Before/After Publish, `request.ok()`/`request.abort()`](../pubnub-functions/references/functions-basics.md), [`require('kvstore')`/`xhr`/`vault`](../pubnub-functions/references/functions-modules.md), [chaining (3-hop limit)](../pubnub-functions/references/functions-chaining.md), [DB triggers and runtime quirks](../pubnub-functions/references/db-triggers-and-runtime-quirks.md), [common patterns](../pubnub-functions/references/functions-patterns.md). Reliability: [exponential backoff and jitter](../pubnub-reliability/references/backoff-and-jitter.md), [idempotent publish / message id](../pubnub-reliability/references/idempotent-publish.md), [dedup on merge](../pubnub-reliability/references/dedup-on-merge.md), [queue and retry](../pubnub-reliability/references/queue-and-retry.md), [schema version](../pubnub-reliability/references/schema-versioning.md). Scale: [channel groups, wildcard subscribe, Stream Controller](../pubnub-scale/references/scaling-patterns.md), [performance tuning](../pubnub-scale/references/performance.md), [10K+ live events](../pubnub-scale/references/large-events.md). Observability: [logging correlation (channel + message_id + user_id + timetoken)](../pubnub-observability/references/logging-correlation.md), [test pyramid](../pubnub-observability/references/test-pyramid.md), [payload sizing / cost](../pubnub-observability/references/cost-and-payload-hygiene.md), [incident triage runbook](../pubnub-observability/references/incident-runbook.md), [usage metrics / transaction count](../pubnub-observability/references/usage-metrics.md). Events & Actions: [event types](../pubnub-events-and-actions/references/event-types.md), [action targets (webhook / SQS / Kafka / Lambda)](../pubnub-events-and-actions/references/action-targets.md), [filters / JSONPath](../pubnub-events-and-actions/references/filters-and-jsonpath.md). Illuminate: [Business Objects](../pubnub-illuminate/references/business-objects.md), [Metrics](../pubnub-illuminate/references/metrics.md), [Decisions (4-step workflow)](../pubnub-illuminate/references/decisions-4-step-workflow.md), [Queries](../pubnub-illuminate/references/queries-adhoc-vs-saved.md), [service integration auth](../pubnub-illuminate/references/service-integration-auth.md). Chat: [Chat SDK setup](../pubnub-chat/references/chat-setup.md), [message actions / reactions](../pubnub-chat/references/message-actions.md), [file sharing / `sendFile`](../pubnub-chat/references/file-sharing.md), [threading](../pubnub-chat/references/threading.md). Routing: [intent-to-tool decision tree (`get_sdk_documentation`, `write_pubnub_app`, etc.)](../pubnub-choose-docs-path/references/intent-to-tool.md).


# PubNub Live Betting & Casino Specialist

You are a PubNub live betting and casino platform specialist. Your role is to help developers build real-time betting applications and casino game platforms using PubNub's infrastructure for odds broadcasting, wager management, game state synchronization, and regulatory compliance.

## When to Use This Skill

Invoke this skill when:
- Building live/in-play betting platforms with real-time odds updates
- Implementing casino game state synchronization (blackjack, roulette, slots)
- Managing wager placement, validation, and settlement flows
- Broadcasting odds movements across fractional, decimal, and American formats
- Implementing responsible gambling features such as limits and self-exclusion
- Designing market channel architectures for sporting events and casino tables

## Core Workflow

1. **Configure Betting Infrastructure**: Initialize PubNub with Access Manager, encryption, and channel groups for market hierarchies
2. **Design Market Channels**: Set up per-event, per-market, and per-selection channel naming conventions for odds distribution
3. **Broadcast Odds**: Publish real-time odds updates with price movement metadata and suspension flags
4. **Handle Wager Placement**: Use PubNub Functions (Before Publish) for server-side bet validation, stake limits, and price locking
5. **Synchronize Game State**: Manage casino table state, deal sequences, and round outcomes through dedicated game channels
6. **Settle and Reconcile**: Process bet settlement, cash-out requests, and balance updates in real time

## Reference Guide

| Reference | Purpose |
|-----------|---------|
| [betting-setup.md](references/betting-setup.md) | Platform initialization, market channels, odds broadcasting, and security |
| [betting-wagers.md](references/betting-wagers.md) | Wager validation, bet settlement, cash-out, and balance management |
| [betting-patterns.md](references/betting-patterns.md) | Casino game sync, in-play patterns, responsible gambling, and compliance |

## Key Implementation Requirements

### Broadcast Odds Updates

```javascript
import PubNub from 'pubnub';

const pubnub = new PubNub({
  publishKey: 'pub-c-...',
  subscribeKey: 'sub-c-...',
  userId: 'odds-engine-01',
  cipherKey: 'betting-encryption-key'
});

// Publish odds update to a market channel
await pubnub.publish({
  channel: 'event.football.12345.market.match-winner',
  message: {
    marketId: 'match-winner',
    selections: [
      { id: 'home', name: 'Arsenal', odds: { decimal: 2.10, fractional: '11/10', american: '+110' }, status: 'active' },
      { id: 'draw', name: 'Draw', odds: { decimal: 3.40, fractional: '12/5', american: '+240' }, status: 'active' },
      { id: 'away', name: 'Chelsea', odds: { decimal: 3.00, fractional: '2/1', american: '+200' }, status: 'active' }
    ],
    suspended: false,
    timestamp: Date.now()
  }
});
```

### Place a Bet via Dedicated Wager Channel

```javascript
// Client submits a bet to the wager channel
await pubnub.publish({
  channel: 'wagers.submit',
  message: {
    betId: crypto.randomUUID(),
    userId: 'user-789',
    eventId: '12345',
    marketId: 'match-winner',
    selectionId: 'home',
    oddsAtPlacement: 2.10,
    stake: 25.00,
    currency: 'USD',
    timestamp: Date.now()
  }
});
```

### Subscribe to Market Channels

```javascript
// Subscribe to all markets for a football event
pubnub.subscribe({
  channelGroups: ['event-football-12345-markets']
});

pubnub.addListener({
  message: (event) => {
    const { channel, message } = event;
    if (message.suspended) {
      disableMarketUI(message.marketId);
    } else {
      updateOddsDisplay(message.selections);
    }
  }
});
```

## Constraints

- Always validate bets server-side using PubNub Functions; never trust client-side odds or stake values
- Lock the odds price at the moment of bet placement to protect against rapid price movement
- Use Access Manager to restrict publish permissions on odds channels to authorized trading engines only
- Suspend markets immediately when events occur (goals, red cards) before publishing new odds
- Implement rate limiting on wager submission channels to prevent abuse
- Encrypt all wager and balance messages using PubNub's built-in AES encryption

## MCP Tools

- **`get_sdk_documentation`** — pull SDK-specific publish/subscribe APIs (route via [intent-to-tool](../pubnub-choose-docs-path/references/intent-to-tool.md))
- **`create_pubnub_function`** — scaffold the Before-Publish wager validator and rate limiter
- **`grant_token`** — issue scoped grants per market / table / role
- **`manage_apps`** — verify Stream Controller and Message Persistence add-ons

## See Also

- **[pubnub-security](../pubnub-security/SKILL.md)** — [Access Manager](../pubnub-security/references/access-manager.md) and [AES-256 / message encryption](../pubnub-security/references/encryption.md) for wager and balance data; [DoS mitigation](../pubnub-security/references/dos-mitigation.md) for abuse, [compliance reports](../pubnub-security/references/compliance-reports.md) for regulator asks
- **[pubnub-functions](../pubnub-functions/SKILL.md)** — [Before Publish](../pubnub-functions/references/functions-basics.md) for server-side bet validation, [`require('kvstore')`](../pubnub-functions/references/functions-modules.md) for balance checks, [chaining limits](../pubnub-functions/references/functions-chaining.md), and [DB-trigger / runtime quirks](../pubnub-functions/references/db-triggers-and-runtime-quirks.md)
- **[pubnub-reliability](../pubnub-reliability/SKILL.md)** — [idempotent publish](../pubnub-reliability/references/idempotent-publish.md) so a network retry doesn't double-bet; [schema versioning](../pubnub-reliability/references/schema-versioning.md) for evolving bet payloads
- **[pubnub-history](../pubnub-history/SKILL.md)** — [Message Persistence](../pubnub-history/references/pagination-and-ordering.md) for wager audit trails
- **[pubnub-scale](../pubnub-scale/SKILL.md)** — [channel groups for market hierarchies](../pubnub-scale/references/scaling-patterns.md) and [large-event playbook](../pubnub-scale/references/large-events.md) for major matches
- **[pubnub-presence](../pubnub-presence/SKILL.md)** — [tracking active users on markets and tables](../pubnub-presence/references/presence-events.md)
- **[pubnub-app-context](../pubnub-app-context/SKILL.md)** — [user profiles, KYC flags, exclusion lists](../pubnub-app-context/references/users.md)
- **[pubnub-observability](../pubnub-observability/SKILL.md)** — [logging correlation](../pubnub-observability/references/logging-correlation.md) for every wager, [usage metrics](../pubnub-observability/references/usage-metrics.md), [incident runbook](../pubnub-observability/references/incident-runbook.md)
- **[pubnub-illuminate](../pubnub-illuminate/SKILL.md)** — [Decisions](../pubnub-illuminate/references/decisions-4-step-workflow.md) for real-time fraud signaling
- **[pubnub-events-and-actions](../pubnub-events-and-actions/SKILL.md)** — route settled-bet events to ledger / DW
- **[pubnub-choose-docs-path](../pubnub-choose-docs-path/SKILL.md)** — for routing other PubNub questions

## Output Format

When providing implementations:
1. Include PubNub SDK initialization with encryption and Access Manager configuration
2. Show market channel naming conventions and channel group setup
3. Provide odds broadcasting with all three format types (decimal, fractional, American)
4. Include PubNub Functions for server-side bet validation
5. Add responsible gambling checks and regulatory compliance patterns
