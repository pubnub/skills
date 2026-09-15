# PubNub Live Sport Updates Setup

## SDK Initialization

One JavaScript publisher is enough here. Pull other-language constructors via **`get_sdk_documentation`**.

```javascript
import PubNub from 'pubnub';

const pubnub = new PubNub({
  publishKey: process.env.PUBNUB_PUBLISH_KEY,
  subscribeKey: process.env.PUBNUB_SUBSCRIBE_KEY,
  secretKey: process.env.PUBNUB_SECRET_KEY, // Server-side only
  userId: 'score-ingestion-service'
});
```

Clients should omit `secretKey`, set `restore: true`, and treat themselves as subscribe-only.

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

Keep a compact envelope. Do not embed sport-rule encyclopedias (down-and-distance, inning lines, goal lists) in the skill — put those fields in your ingestion mapper.

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
    label: 'string',          // Display label from the provider feed
    clock: 'string'           // Game clock from the provider feed
  }
};
```

## Publishing Score Updates

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
    // Reconnect: wire status per [S3 backoff-and-jitter](../../pubnub-reliability/references/backoff-and-jitter.md);
    // on reconnect run [S2 offline catch-up](../../pubnub-history/references/offline-catch-up.md) — see Reconnection section below.
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

Follow [offline catch-up](../../pubnub-history/references/offline-catch-up.md) with [dedup-on-merge](../../pubnub-reliability/references/dedup-on-merge.md). **Sport-specific:** after reconnect, fetch missed messages for the game channels and dispatch `score_update` / `play_by_play` / `game_status` to the same handlers — dedupe by `gameId + sequence`.

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
4. **Reconnection** — Enable `restore: true` and `autoNetworkDetection: true`; on `PNReconnectedCategory` run [offline catch-up](../../pubnub-history/references/offline-catch-up.md) (see Reconnection section above)
5. **Wildcard subscriptions** - Design channel names to support wildcards at meaningful boundaries (league, team, game)
6. **Access control** - Use Access Manager to restrict publish rights to your ingestion service; clients should be subscribe-only
7. **Idempotent processing** - Clients should deduplicate by gameId + sequence to handle redelivery gracefully
8. **Time source** - Use server-side timestamps exclusively; never rely on client clocks for event ordering
9. **Cleanup** - Always unsubscribe and destroy the PubNub instance when the user navigates away
