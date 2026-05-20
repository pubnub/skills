<!-- xrefs-injected -->

> **Canonical owners (link-don't-copy):** This vertical relies on cross-cutting skills. Always link to the canonical owner instead of duplicating. Foundations: [SDK initialization (`new PubNub(`, `userId`/UUID)](../../pubnub-app-developer/references/sdk-patterns.md), [pub/sub basics (`pubnub.publish(`, `pubnub.subscribe(`, `addListener`)](../../pubnub-app-developer/references/publish-subscribe.md), [channel naming](../../pubnub-app-developer/references/channels.md), [message filters](../../pubnub-app-developer/references/message-filters.md), [SDK upgrades](../../pubnub-app-developer/references/sdk-upgrades.md), [REST API](../../pubnub-app-developer/references/rest-api.md). Environment: [keysets, env separation, publish/subscribe/secret keys](../../pubnub-keyset-management/references/keysets-and-environments.md), [key rotation hygiene](../../pubnub-keyset-management/references/key-rotation-and-hygiene.md), [demo keys](../../pubnub-keyset-management/references/demo-keys.md), [custom origin](../../pubnub-keyset-management/references/custom-origin.md). Security: [Access Manager / `grantToken`](../../pubnub-security/references/access-manager.md), [AES-256 / message encryption](../../pubnub-security/references/encryption.md), [IP allowlisting](../../pubnub-security/references/ip-whitelisting.md), [DoS mitigation](../../pubnub-security/references/dos-mitigation.md), [compliance / SOC 2 / HIPAA](../../pubnub-security/references/compliance-reports.md). Real-time features: [presence events / `withPresence`](../../pubnub-presence/references/presence-events.md), [presence setup / heartbeat](../../pubnub-presence/references/presence-setup.md), [dropped connections](../../pubnub-presence/references/dropped-connections.md), [multi-device sync](../../pubnub-presence/references/multi-device-sync.md). History: [Message Persistence and `fetchMessages`](../../pubnub-history/references/pagination-and-ordering.md), [offline catch-up](../../pubnub-history/references/offline-catch-up.md), [retention](../../pubnub-history/references/retention-and-storage.md). App Context: [users / user metadata](../../pubnub-app-context/references/users.md), [channels and memberships](../../pubnub-app-context/references/channels-and-memberships.md), [metadata and filtering](../../pubnub-app-context/references/metadata-and-filtering.md). Functions: [Before/After Publish, `request.ok()`/`request.abort()`](../../pubnub-functions/references/functions-basics.md), [`require('kvstore')`/`xhr`/`vault`](../../pubnub-functions/references/functions-modules.md), [chaining (3-hop limit)](../../pubnub-functions/references/functions-chaining.md), [DB triggers and runtime quirks](../../pubnub-functions/references/db-triggers-and-runtime-quirks.md), [common patterns](../../pubnub-functions/references/functions-patterns.md). Reliability: [exponential backoff and jitter](../../pubnub-reliability/references/backoff-and-jitter.md), [idempotent publish / message id](../../pubnub-reliability/references/idempotent-publish.md), [dedup on merge](../../pubnub-reliability/references/dedup-on-merge.md), [queue and retry](../../pubnub-reliability/references/queue-and-retry.md), [schema version](../../pubnub-reliability/references/schema-versioning.md). Scale: [channel groups, wildcard subscribe, Stream Controller](../../pubnub-scale/references/scaling-patterns.md), [performance tuning](../../pubnub-scale/references/performance.md), [10K+ live events](../../pubnub-scale/references/large-events.md). Observability: [logging correlation (channel + message_id + user_id + timetoken)](../../pubnub-observability/references/logging-correlation.md), [test pyramid](../../pubnub-observability/references/test-pyramid.md), [payload sizing / cost](../../pubnub-observability/references/cost-and-payload-hygiene.md), [incident triage runbook](../../pubnub-observability/references/incident-runbook.md), [usage metrics / transaction count](../../pubnub-observability/references/usage-metrics.md). Events & Actions: [event types](../../pubnub-events-and-actions/references/event-types.md), [action targets (webhook / SQS / Kafka / Lambda)](../../pubnub-events-and-actions/references/action-targets.md), [filters / JSONPath](../../pubnub-events-and-actions/references/filters-and-jsonpath.md). Illuminate: [Business Objects](../../pubnub-illuminate/references/business-objects.md), [Metrics](../../pubnub-illuminate/references/metrics.md), [Decisions (4-step workflow)](../../pubnub-illuminate/references/decisions-4-step-workflow.md), [Queries](../../pubnub-illuminate/references/queries-adhoc-vs-saved.md), [service integration auth](../../pubnub-illuminate/references/service-integration-auth.md). Chat: [Chat SDK setup](../../pubnub-chat/references/chat-setup.md), [message actions / reactions](../../pubnub-chat/references/message-actions.md), [file sharing / `sendFile`](../../pubnub-chat/references/file-sharing.md), [threading](../../pubnub-chat/references/threading.md). Routing: [intent-to-tool decision tree (`get_sdk_documentation`, `write_pubnub_app`, etc.)](../../pubnub-choose-docs-path/references/intent-to-tool.md).


# PubNub Live Sport Updates Setup

## Installation

### JavaScript/TypeScript

```bash
npm install pubnub
# or
yarn add pubnub
```

### Prerequisites

- Node.js >= 16.0.0
- Publish and Subscribe keys from PubNub Admin Portal
- **Message Persistence** enabled for historical score lookups
- **Stream Controller** enabled for wildcard subscriptions

## SDK Initialization

### Score Ingestion Service (Server-Side)

```javascript
import PubNub from 'pubnub';

const pubnub = new PubNub({
  publishKey: process.env.PUBNUB_PUBLISH_KEY,
  subscribeKey: process.env.PUBNUB_SUBSCRIBE_KEY,
  secretKey: process.env.PUBNUB_SECRET_KEY, // Server-side only
  userId: 'score-ingestion-service'
});
```

### Client Application (Browser/Mobile)

```javascript
import PubNub from 'pubnub';

const pubnub = new PubNub({
  subscribeKey: 'sub-c-...',
  userId: `fan-${currentUserId}`,
  restore: true,           // Reconnect and catch up on missed messages
  autoNetworkDetection: true,
  heartbeatInterval: 30
});
```

### React Integration

```javascript
import { useState, useEffect } from 'react';
import PubNub from 'pubnub';

function SportsApp({ userId }) {
  const [pubnub, setPubnub] = useState(null);

  useEffect(() => {
    const pn = new PubNub({
      subscribeKey: process.env.REACT_APP_PUBNUB_SUB_KEY,
      userId: `fan-${userId}`,
      restore: true,
      autoNetworkDetection: true
    });
    setPubnub(pn);

    return () => {
      pn.unsubscribeAll();
      pn.destroy();
    };
  }, [userId]);

  if (!pubnub) return <div>Loading...</div>;
  return <ScoreboardDashboard pubnub={pubnub} />;
}
```

## Channel Hierarchy

The channel naming convention follows a dot-delimited hierarchy that supports wildcard subscriptions at every level.

### Channel Naming Pattern

```
sports.<league>.<context>.<identifier>
```

### Channel Types

| Channel Pattern | Purpose | Example |
|----------------|---------|---------|
| `sports.<league>.scores` | League-wide score ticker | `sports.nfl.scores` |
| `sports.<league>.games.<gameId>` | Single game updates | `sports.nfl.games.2024-SEA-SF-week5` |
| `sports.<league>.games.<gameId>.plays` | Play-by-play for one game | `sports.nfl.games.2024-SEA-SF-week5.plays` |
| `sports.<league>.teams.<teamId>` | All updates for one team | `sports.nfl.teams.SF` |
| `sports.<league>.standings` | Standings and league table | `sports.epl.standings` |
| `sports.<league>.games.<gameId>.fan` | Fan engagement for a game | `sports.nba.games.2024-LAL-BOS-g3.fan` |

### Wildcard Subscription Examples

```javascript
// All NFL game updates
pubnub.subscribe({ channels: ['sports.nfl.games.*'] });

// All NBA games and standings
pubnub.subscribe({
  channels: ['sports.nba.games.*', 'sports.nba.standings']
});
```

### Sport-Specific Channel Tables

#### American Football (NFL)

| Channel | Content |
|---------|---------|
| `sports.nfl.games.<gameId>` | Score updates, quarter changes, game status |
| `sports.nfl.games.<gameId>.plays` | Individual plays, penalties, challenges |
| `sports.nfl.redzone` | Aggregated red zone alerts across all games |

#### Basketball (NBA)

| Channel | Content |
|---------|---------|
| `sports.nba.games.<gameId>` | Score updates, quarter changes |
| `sports.nba.games.<gameId>.plays` | Shot attempts, assists, turnovers |
| `sports.nba.scores` | All active game scores |

#### Soccer (EPL, MLS, UEFA)

| Channel | Content |
|---------|---------|
| `sports.epl.games.<gameId>` | Score updates, half changes |
| `sports.epl.games.<gameId>.plays` | Shots, fouls, cards, substitutions |
| `sports.epl.standings` | League table updates |

#### Baseball (MLB)

| Channel | Content |
|---------|---------|
| `sports.mlb.games.<gameId>` | Score updates, inning changes |
| `sports.mlb.games.<gameId>.plays` | At-bats, pitches, base running |

## Score Data Models

### Universal Game State

```javascript
const gameState = {
  gameId: 'string',           // Unique game identifier
  sport: 'string',            // nfl, nba, mlb, epl, nhl
  status: 'string',           // pre_game | in_progress | halftime | delayed | final
  timestamp: 'number',        // Server-side Unix ms timestamp
  sequence: 'number',         // Monotonically increasing per game
  home: { team: 'string', name: 'string', score: 'number' },
  away: { team: 'string', name: 'string', score: 'number' },
  period: {
    current: 'number',        // Period number (1-based)
    label: 'string',          // Display label (Q1, 1st Half, Top 3rd)
    clock: 'string'           // Game clock (mm:ss or empty)
  }
};
```

### NFL Score Model

```javascript
const nflScore = {
  sport: 'nfl',
  period: { current: 3, label: 'Q3', clock: '07:42' },
  home: { team: 'SF', name: '49ers', score: 21 },
  away: { team: 'SEA', name: 'Seahawks', score: 14 },
  scoring: { home: { q1: 7, q2: 7, q3: 7, q4: 0 }, away: { q1: 0, q2: 7, q3: 7, q4: 0 } },
  possession: 'SF',
  down: 2,
  yardsToGo: 7,
  yardLine: 'SEA 35'
};
```

### Soccer Score Model

```javascript
const soccerScore = {
  sport: 'epl',
  period: { current: 2, label: '2nd Half', clock: '72:15' },
  home: { team: 'ARS', name: 'Arsenal', score: 2 },
  away: { team: 'CHE', name: 'Chelsea', score: 1 },
  goals: [
    { team: 'ARS', player: 'B. Saka', minute: 23, type: 'open_play' },
    { team: 'CHE', player: 'C. Palmer', minute: 41, type: 'penalty' },
    { team: 'ARS', player: 'K. Havertz', minute: 68, type: 'header' }
  ],
  cards: { home: { yellow: 1, red: 0 }, away: { yellow: 2, red: 0 } }
};
```

### MLB Score Model

```javascript
const mlbScore = {
  sport: 'mlb',
  period: { current: 7, label: 'Bot 7th', clock: '' },
  home: { team: 'NYY', name: 'Yankees', score: 5 },
  away: { team: 'BOS', name: 'Red Sox', score: 3 },
  inningScores: {
    home: [0, 1, 0, 2, 0, 0, 2, null, null],
    away: [1, 0, 0, 0, 2, 0, 0, null, null]
  },
  count: { balls: 2, strikes: 1, outs: 1 },
  bases: { first: true, second: false, third: true }
};
```

## Data Ingestion from Sports Providers

```javascript
class SportDataIngestionService {
  constructor(pubnub) {
    this.pubnub = pubnub;
    this.sequenceCounters = new Map();
  }

  getNextSequence(gameId) {
    const next = (this.sequenceCounters.get(gameId) || 0) + 1;
    this.sequenceCounters.set(gameId, next);
    return next;
  }

  async publishScoreUpdate(game) {
    const channel = `sports.${game.league}.games.${game.gameId}`;
    const sequence = this.getNextSequence(game.gameId);

    await this.pubnub.publish({
      channel,
      message: { type: 'score_update', sequence, timestamp: Date.now(), ...game }
    });

    // Also publish summary to league scores channel
    await this.pubnub.publish({
      channel: `sports.${game.league}.scores`,
      message: {
        type: 'score_summary',
        gameId: game.gameId,
        home: { team: game.home.team, score: game.home.score },
        away: { team: game.away.team, score: game.away.score },
        status: game.status,
        period: game.period
      }
    });
  }

  normalizeEvent(rawEvent) {
    return {
      gameId: rawEvent.id,
      league: rawEvent.sport_code.toLowerCase(),
      sport: rawEvent.sport_code.toLowerCase(),
      status: this.mapStatus(rawEvent.state),
      home: { team: rawEvent.home_team.abbreviation, name: rawEvent.home_team.name, score: rawEvent.home_score },
      away: { team: rawEvent.away_team.abbreviation, name: rawEvent.away_team.name, score: rawEvent.away_score },
      period: { current: rawEvent.period, label: rawEvent.period_label, clock: rawEvent.clock || '' }
    };
  }

  mapStatus(providerStatus) {
    const statusMap = { 'scheduled': 'pre_game', 'in_progress': 'in_progress', 'halftime': 'halftime', 'delayed': 'delayed', 'final': 'final' };
    return statusMap[providerStatus] || 'unknown';
  }
}
```

## Subscription Patterns

### Following a Single Game

```javascript
function subscribeToGame(pubnub, league, gameId, handlers) {
  const channels = [
    `sports.${league}.games.${gameId}`,
    `sports.${league}.games.${gameId}.plays`
  ];

  const listener = {
    message: (event) => {
      switch (event.message.type) {
        case 'score_update': handlers.onScoreUpdate?.(event.message); break;
        case 'play_by_play': handlers.onPlay?.(event.message); break;
        case 'game_status': handlers.onStatusChange?.(event.message); break;
      }
    },
    status: (event) => {
      if (event.category === 'PNReconnectedCategory') {
        fetchMissedUpdates(pubnub, channels, handlers);
      }
    }
  };

  pubnub.addListener(listener);
  pubnub.subscribe({ channels });

  return () => {
    pubnub.removeListener(listener);
    pubnub.unsubscribe({ channels });
  };
}
```

### Reconnection and Catch-Up

```javascript
async function fetchMissedUpdates(pubnub, channels, handlers) {
  for (const channel of channels) {
    try {
      const response = await pubnub.fetchMessages({ channels: [channel], count: 25 });
      const messages = response.channels[channel] || [];
      for (const entry of messages) {
        if (entry.message.type === 'score_update') {
          handlers.onScoreUpdate?.(entry.message);
        }
      }
    } catch (error) {
      console.error(`Failed to fetch missed updates for ${channel}:`, error);
    }
  }
}
```

## Mobile SDK Initialization

### Swift (iOS)

```swift
import PubNub

let config = PubNubConfiguration(
    publishKey: "pub-c-...",
    subscribeKey: "sub-c-...",
    userId: "fan-ios-\(userId)"
)
let pubnub = PubNub(configuration: config)
pubnub.subscribe(to: ["sports.nfl.games.*"])
```

### Kotlin (Android)

```kotlin
val config = PNConfiguration(UserId("fan-android-$userId")).apply {
    subscribeKey = "sub-c-..."
}
val pubnub = PubNub.create(config)
pubnub.subscribe(channels = listOf("sports.nfl.games.*"))
```

## Required Keyset Settings

| Setting | Required | Purpose |
|---------|----------|---------|
| Message Persistence | Yes | Historical scores and replay |
| Stream Controller | Yes | Wildcard channel subscriptions |
| Access Manager | Recommended | Restrict publish to ingestion service |
| PubNub Functions | Optional | Server-side event enrichment |
| Push Notifications | Optional | Mobile push for key events |

## Best Practices

1. **Channel granularity** - Use separate channels for scores, play-by-play, and fan engagement so clients subscribe only to what they render
2. **Compact payloads** - Keep real-time messages under 2 KB; use abbreviations and codes rather than full names
3. **Sequence numbers** - Always include a per-game monotonic sequence so clients detect gaps and request backfill
4. **Reconnection** - Enable `restore: true` and `autoNetworkDetection: true`; fetch missed messages on reconnect
5. **Wildcard subscriptions** - Design channel names to support wildcards at meaningful boundaries (league, team, game)
6. **Access control** - Use Access Manager to restrict publish rights to your ingestion service; clients should be subscribe-only
7. **Idempotent processing** - Clients should deduplicate by gameId + sequence to handle redelivery gracefully
8. **Time source** - Use server-side timestamps exclusively; never rely on client clocks for event ordering
9. **Cleanup** - Always unsubscribe and destroy the PubNub instance when the user navigates away
