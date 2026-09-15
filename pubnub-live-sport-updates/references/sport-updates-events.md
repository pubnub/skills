# PubNub Sport Game Events

## Event Type Taxonomy

Every game event published to PubNub follows a common envelope with a sport-specific payload. Events are categorized into three tiers based on impact and urgency.

### Event Tiers

| Tier | Description | Examples | Delivery Priority |
|------|-------------|----------|-------------------|
| Critical | Scoring plays and game status changes | Goals, touchdowns, game start/end | Immediate + push notification |
| Standard | Significant in-game actions | Fouls, substitutions, timeouts | Immediate |
| Informational | Context and statistics | Possession changes, stat updates | Batched (1-5 second window) |

Map provider play types in your ingestion layer. Do not keep sport-rule encyclopedias (NFL/NBA/Soccer/MLB scoring catalogs, period clocks) in this skill.

## Universal Event Envelope

```javascript
const gameEvent = {
  type: 'string',            // Event type identifier
  gameId: 'string',          // Unique game identifier
  sport: 'string',           // nfl, nba, mlb, epl, nhl
  sequence: 'number',        // Per-game monotonic sequence number
  timestamp: 'number',       // Server-side Unix ms timestamp
  period: {
    current: 'number',
    label: 'string',
    clock: 'string'
  },
  score: {
    home: { team: 'string', score: 'number' },
    away: { team: 'string', score: 'number' }
  },
  payload: {}                // Sport-specific event data from the provider
};
```

## Publishing Game Events

### Event Publisher Service

```javascript
class GameEventPublisher {
  constructor(pubnub) {
    this.pubnub = pubnub;
    this.sequenceMap = new Map();
  }

  nextSequence(gameId) {
    const seq = (this.sequenceMap.get(gameId) || 0) + 1;
    this.sequenceMap.set(gameId, seq);
    return seq;
  }

  async publishEvent(event) {
    const sequence = this.nextSequence(event.gameId);
    const enrichedEvent = { ...event, sequence, timestamp: Date.now() };

    // 3-level max: sports.<league>.<gameId> and sports.<league>.plays_<gameId>
    const gameChannel = `sports.${event.sport}.${event.gameId}`;
    const playsChannel = `sports.${event.sport}.plays_${event.gameId}`;

    if (this.isCriticalEvent(event.type, event.sport)) {
      await Promise.all([
        this.pubnub.publish({ channel: gameChannel, message: enrichedEvent }),
        this.pubnub.publish({ channel: playsChannel, message: enrichedEvent })
      ]);
      await this.pubnub.publish({
        channel: `sports.${event.sport}.scores`,
        message: { type: 'score_summary', gameId: event.gameId, score: event.score, period: event.period }
      });
    } else {
      await this.pubnub.publish({ channel: playsChannel, message: enrichedEvent });
    }
  }

  isCriticalEvent(type, sport) {
    const criticalEvents = {
      nfl: ['touchdown', 'field_goal', 'safety', 'game_status'],
      nba: ['three_pointer', 'dunk', 'game_status'],
      epl: ['goal', 'red_card', 'penalty_awarded', 'game_status'],
      mlb: ['home_run', 'run_scored', 'game_status'],
      nhl: ['goal', 'game_status']
    };
    return (criticalEvents[sport] || []).includes(type);
  }
}
```

### Publishing Game Status Transitions

```javascript
async function publishGameStatus(publisher, gameId, sport, newStatus, period, score) {
  await publisher.publishEvent({
    type: 'game_status',
    gameId,
    sport,
    period,
    score,
    payload: { status: newStatus }
  });
}
```

## Standings and League Tables

Publish already-ranked standings. Do not implement league-table math (points, GD, sort keys) in this skill.

```javascript
async function publishStandings(pubnub, league, standings) {
  await pubnub.publish({
    channel: `sports.${league}.standings`,
    message: {
      type: 'standings_update',
      league,
      timestamp: Date.now(),
      standings
    }
  });
}
```

## Play-by-Play Feed Construction

Use [delta / sequence state synchronization](../../pubnub-multiplayer-gaming/references/gaming-state-sync.md) for monotonic sequence tracking, gap detection, and backfill. **Sport-specific delta:** maintain a play-by-play timeline sorted by `sequence`; on gap, request backfill for the missing range; filter scoring events by the provider’s play types.

## Error Handling

### Out-of-Order Event Recovery

The backfill inside this handler applies the canonical [offline catch-up pattern (S2)](../../pubnub-history/references/offline-catch-up.md). **Sport-specific delta:** target the single game channel (`sports.{league}.{gameId}`), merge via `timeline.addEvent`, and retry with [exponential backoff (S3)](../../pubnub-reliability/references/backoff-and-jitter.md) on history failure.

```javascript
async function handleEventWithRecovery(pubnub, event, timeline, gameChannel) {
  try {
    timeline.addEvent(event);
  } catch (error) {
    console.error('Failed to process event:', error);
  }

  if (timeline.pendingOutOfOrder.length > 0) {
    try {
      const history = await pubnub.fetchMessages({ channels: [gameChannel], count: 100 });
      const messages = history.channels[gameChannel] || [];
      for (const msg of messages) { timeline.addEvent(msg.message); }
    } catch (historyError) {
      console.error('Backfill failed:', historyError);
      scheduleRetry(() => handleEventWithRecovery(pubnub, event, timeline, gameChannel));
    }
  }
}

function scheduleRetry(fn, attempt = 1, maxAttempts = 5) {
  if (attempt > maxAttempts) return;
  const delay = Math.min(1000 * Math.pow(2, attempt - 1), 30000);
  setTimeout(() => fn(), delay);
}
```

### Event Validation

```javascript
function validateGameEvent(event) {
  const errors = [];
  if (!event.type) errors.push('Missing event type');
  if (!event.gameId) errors.push('Missing gameId');
  if (!event.sport) errors.push('Missing sport');
  if (typeof event.sequence !== 'number') errors.push('Invalid sequence');
  if (!event.score?.home || !event.score?.away) errors.push('Missing score data');
  if (typeof event.period?.current !== 'number') errors.push('Invalid period');

  if (errors.length > 0) {
    console.error('Invalid game event:', errors.join(', '));
    return false;
  }
  return true;
}
```

## Best Practices

1. **Event ordering** - Always include a per-game monotonically increasing sequence number; never rely on network delivery order alone
2. **Tier-based routing** - Publish critical events to both the game channel and plays channel; informational events to plays only
3. **Idempotent clients** - Deduplicate by gameId + sequence so redelivered messages do not corrupt the timeline
4. **Compact payloads** - Include only delta information; clients maintain local game state and apply updates incrementally
5. **Status transitions** - Publish explicit game_status events for every state change so clients do not infer state from scores
6. **Clock synchronization** - Always use the server-side game clock value; never derive it from wall-clock time
7. **Backfill on reconnect** - Fetch recent history and replay events through the timeline builder to close gaps
8. **Validation at ingestion** - Validate every event before publishing; reject malformed events to protect downstream consumers
9. **Scoring summary** - Maintain a separate list of scoring plays for quick catch-up when users join mid-game
