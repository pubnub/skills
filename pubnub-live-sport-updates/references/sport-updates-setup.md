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

> **Wildcard Subscribe constraint:** Wildcard patterns are limited to **two dots** (`sports.*` or `sports.nba.*`, not `sports.nba.games.*`). Plain channel names are **not** capped at three segments — but if you rely on wildcards, design names so a valid pattern covers your leaves.

The channel naming convention uses a dot-delimited hierarchy. When wildcards are required, prefer three-segment leaves under a two-dot pattern (e.g. `sports.<league>.<segment>` with `sports.<league>.*`). For deeper paths like `sports.<league>.games.<gameId>`, use explicit subscribe, channel groups, or encode extra dimensions in the segment (`plays_<gameId>`).

### Channel Naming Pattern

```
sports.<league>.<segment>
```

`<segment>` is the game ID, team ID, or a fixed keyword. When a game needs multiple event-type channels, put the type as a **prefix** before the game ID using an underscore: `plays_<gameId>`, `fan_<gameId>`. This keeps the type at the front of the segment where Functions channel binding patterns (e.g. `sports.nfl.plays*`) can leverage it.

### Channel Types

| Channel Pattern | Purpose | Example |
|----------------|---------|---------|
| `sports.<league>.scores` | League-wide score ticker | `sports.nfl.scores` |
| `sports.<league>.standings` | Standings and league table | `sports.epl.standings` |
| `sports.<league>.<gameId>` | All updates for a single game | `sports.nfl.2024-SEA-SF-wk5` |
| `sports.<league>.plays_<gameId>` | Play-by-play for one game | `sports.nfl.plays_2024-SEA-SF-wk5` |
| `sports.<league>.fan_<gameId>` | Fan engagement for a game | `sports.nba.fan_2024-LAL-BOS-g3` |
| `sports.<league>.team_<teamId>` | All updates for one team | `sports.nfl.team_SF` |

### Wildcard Subscription Examples

```javascript
// All NFL channels (scores, standings, all games)
pubnub.subscribe({ channels: ['sports.nfl.*'] });

// All NBA channels
pubnub.subscribe({
  channels: ['sports.nba.*']
});
```

> Note: with the 3-level design you can wildcard-subscribe to an entire league (`sports.nfl.*`) but not to a specific game across event types — that would require 4 levels. If per-game wildcard subscription is critical, put all game event types on a single channel and use a `type` field in the message payload to differentiate them.

### Sport-Specific Channel Tables

#### American Football (NFL)

| Channel | Content |
|---------|---------|
| `sports.nfl.<gameId>` | Score updates, quarter changes, game status |
| `sports.nfl.plays_<gameId>` | Individual plays, penalties, challenges |
| `sports.nfl.redzone` | Aggregated red zone alerts across all games |

#### Basketball (NBA)

| Channel | Content |
|---------|---------|
| `sports.nba.<gameId>` | Score updates, quarter changes |
| `sports.nba.plays_<gameId>` | Shot attempts, assists, turnovers |
| `sports.nba.scores` | All active game scores |

#### Soccer (EPL, MLS, UEFA)

| Channel | Content |
|---------|---------|
| `sports.epl.<gameId>` | Score updates, half changes |
| `sports.epl.plays_<gameId>` | Shots, fouls, cards, substitutions |
| `sports.epl.standings` | League table updates |

#### Baseball (MLB)

| Channel | Content |
|---------|---------|
| `sports.mlb.<gameId>` | Score updates, inning changes |
| `sports.mlb.plays_<gameId>` | At-bats, pitches, base running |

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
    // 3-level max: sports.<league>.<gameId>  — do NOT add a 4th segment
    const channel = `sports.${game.league}.${game.gameId}`;
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
  // 3-level max: sports.<league>.<gameId> and sports.<league>.plays_<gameId>
  const channels = [
    `sports.${league}.${gameId}`,
    `sports.${league}.plays_${gameId}`
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
pubnub.subscribe(to: ["sports.nfl.*"])
```

### Kotlin (Android)

```kotlin
val config = PNConfiguration(UserId("fan-android-$userId")).apply {
    subscribeKey = "sub-c-..."
}
val pubnub = PubNub.create(config)
pubnub.subscribe(channels = listOf("sports.nfl.*"))
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
