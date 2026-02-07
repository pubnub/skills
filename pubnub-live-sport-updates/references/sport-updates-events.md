# PubNub Sport Game Events

## Event Type Taxonomy

Every game event published to PubNub follows a common envelope with a sport-specific payload. Events are categorized into three tiers based on impact and urgency.

### Event Tiers

| Tier | Description | Examples | Delivery Priority |
|------|-------------|----------|-------------------|
| Critical | Scoring plays and game status changes | Goals, touchdowns, game start/end | Immediate + push notification |
| Standard | Significant in-game actions | Fouls, substitutions, timeouts | Immediate |
| Informational | Context and statistics | Possession changes, stat updates | Batched (1-5 second window) |

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
  payload: {}                // Sport-specific event data
};
```

## Sport-Specific Event Types

### NFL Event Types

| Event Type | Tier | Description |
|------------|------|-------------|
| `touchdown` | Critical | Touchdown scored |
| `field_goal` | Critical | Field goal made or missed |
| `safety` | Critical | Safety scored |
| `extra_point` | Standard | PAT attempt result |
| `two_point_conversion` | Standard | Two-point conversion result |
| `turnover` | Standard | Interception or fumble recovery |
| `penalty` | Standard | Penalty called |
| `quarter_change` | Standard | Quarter transition |
| `play` | Informational | Individual play result |

```javascript
const touchdownEvent = {
  type: 'touchdown',
  gameId: '2024-SEA-SF-week5',
  sport: 'nfl',
  sequence: 142,
  timestamp: Date.now(),
  period: { current: 3, label: 'Q3', clock: '04:32' },
  score: { home: { team: 'SF', score: 21 }, away: { team: 'SEA', score: 14 } },
  payload: {
    team: 'SF',
    player: 'C. McCaffrey',
    playType: 'rush',
    yards: 12,
    description: 'C. McCaffrey 12 yard rush',
    driveInfo: { plays: 8, yards: 75, timeOfPossession: '4:23' }
  }
};
```

### NBA Event Types

| Event Type | Tier | Description |
|------------|------|-------------|
| `three_pointer` | Critical | Three-point basket made |
| `dunk` | Critical | Dunk (highlight play) |
| `basket` | Standard | Two-point field goal |
| `free_throw` | Standard | Free throw attempt |
| `foul` | Standard | Personal or technical foul |
| `timeout` | Standard | Timeout called |
| `quarter_change` | Standard | Quarter transition |
| `rebound` | Informational | Offensive or defensive rebound |
| `block` | Informational | Shot blocked |

```javascript
const threePointerEvent = {
  type: 'three_pointer',
  gameId: '2024-LAL-BOS-finals-g3',
  sport: 'nba',
  sequence: 312,
  timestamp: Date.now(),
  period: { current: 4, label: 'Q4', clock: '01:45' },
  score: { home: { team: 'BOS', score: 101 }, away: { team: 'LAL', score: 98 } },
  payload: {
    team: 'BOS',
    player: 'J. Tatum',
    distance: 28,
    assisted: true,
    assistPlayer: 'J. Brown'
  }
};
```

### Soccer Event Types

| Event Type | Tier | Description |
|------------|------|-------------|
| `goal` | Critical | Goal scored |
| `red_card` | Critical | Red card issued |
| `penalty_awarded` | Critical | Penalty kick awarded |
| `yellow_card` | Standard | Yellow card issued |
| `substitution` | Standard | Player substituted |
| `half_change` | Standard | Half transition |
| `shot_on_target` | Informational | Shot on goal |
| `corner` | Informational | Corner kick awarded |
| `offside` | Informational | Offside call |

```javascript
const goalEvent = {
  type: 'goal',
  gameId: '2024-ARS-CHE-epl-md12',
  sport: 'epl',
  sequence: 87,
  timestamp: Date.now(),
  period: { current: 2, label: '2nd Half', clock: '68:22' },
  score: { home: { team: 'ARS', score: 2 }, away: { team: 'CHE', score: 1 } },
  payload: {
    team: 'ARS',
    scorer: 'K. Havertz',
    assist: 'B. Saka',
    goalType: 'header',
    minute: 68
  }
};
```

### MLB Event Types

| Event Type | Tier | Description |
|------------|------|-------------|
| `home_run` | Critical | Home run hit |
| `run_scored` | Critical | Run crosses home plate |
| `strikeout` | Standard | Batter struck out |
| `walk` | Standard | Base on balls |
| `hit` | Standard | Base hit (single, double, triple) |
| `inning_change` | Standard | Inning transition |
| `pitch` | Informational | Individual pitch result |
| `out` | Informational | Out recorded |

```javascript
const homeRunEvent = {
  type: 'home_run',
  gameId: '2024-NYY-BOS-aug15',
  sport: 'mlb',
  sequence: 198,
  timestamp: Date.now(),
  period: { current: 5, label: 'Bot 5th', clock: '' },
  score: { home: { team: 'NYY', score: 6 }, away: { team: 'BOS', score: 3 } },
  payload: {
    team: 'NYY',
    batter: 'A. Judge',
    pitcher: 'C. Sale',
    exitVelocity: 112.4,
    distance: 425,
    runsBattedIn: 2,
    description: 'A. Judge 2-run homer to center (425 ft)'
  }
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

    const gameChannel = `sports.${event.sport}.games.${event.gameId}`;
    const playsChannel = `${gameChannel}.plays`;

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
    payload: { status: newStatus, statusMessage: getStatusMessage(newStatus, sport) }
  });
}

function getStatusMessage(status, sport) {
  const messages = {
    'pre_game': 'Game starting soon',
    'in_progress': 'Game underway',
    'halftime': sport === 'epl' ? 'Half-time' : 'Halftime',
    'final': 'Final',
    'delayed': 'Game delayed',
    'overtime': 'Overtime'
  };
  return messages[status] || status;
}
```

## Standings and League Tables

### Publishing Standings Updates

```javascript
async function publishStandings(pubnub, league, standings) {
  await pubnub.publish({
    channel: `sports.${league}.standings`,
    message: {
      type: 'standings_update',
      league,
      timestamp: Date.now(),
      standings: standings.map(entry => ({
        rank: entry.rank, team: entry.team, name: entry.name,
        played: entry.wins + entry.losses + (entry.draws || 0),
        wins: entry.wins, draws: entry.draws || 0, losses: entry.losses,
        points: entry.points, goalDifference: entry.goalDifference
      }))
    }
  });
}
```

### Soccer League Table Calculation

```javascript
function calculateLeagueTable(results) {
  const table = new Map();

  for (const match of results) {
    if (match.status !== 'final') continue;
    const home = getOrCreate(table, match.home.team, match.home.name);
    const away = getOrCreate(table, match.away.team, match.away.name);

    home.played++; away.played++;
    home.goalsFor += match.home.score; home.goalsAgainst += match.away.score;
    away.goalsFor += match.away.score; away.goalsAgainst += match.home.score;

    if (match.home.score > match.away.score) {
      home.wins++; home.points += 3; away.losses++;
    } else if (match.home.score < match.away.score) {
      away.wins++; away.points += 3; home.losses++;
    } else {
      home.draws++; away.draws++; home.points += 1; away.points += 1;
    }
    home.goalDifference = home.goalsFor - home.goalsAgainst;
    away.goalDifference = away.goalsFor - away.goalsAgainst;
  }

  return [...table.values()]
    .sort((a, b) => b.points - a.points || b.goalDifference - a.goalDifference || b.goalsFor - a.goalsFor)
    .map((entry, i) => ({ ...entry, rank: i + 1 }));
}

function getOrCreate(table, team, name) {
  if (!table.has(team)) {
    table.set(team, { team, name, played: 0, wins: 0, draws: 0, losses: 0, goalsFor: 0, goalsAgainst: 0, goalDifference: 0, points: 0 });
  }
  return table.get(team);
}
```

## Play-by-Play Feed Construction

### Client-Side Timeline Builder

```javascript
class PlayByPlayTimeline {
  constructor() {
    this.events = [];
    this.lastSequence = 0;
    this.pendingOutOfOrder = [];
  }

  addEvent(event) {
    if (event.sequence <= this.lastSequence) {
      if (this.events.some(e => e.sequence === event.sequence)) return;
      this.insertSorted(event);
      return;
    }
    if (event.sequence > this.lastSequence + 1) {
      this.pendingOutOfOrder.push(event);
      this.requestBackfill(this.lastSequence + 1, event.sequence - 1);
      return;
    }
    this.events.push(event);
    this.lastSequence = event.sequence;
    this.processPending();
  }

  insertSorted(event) {
    const index = this.events.findIndex(e => e.sequence > event.sequence);
    if (index === -1) this.events.push(event);
    else this.events.splice(index, 0, event);
  }

  processPending() {
    this.pendingOutOfOrder.sort((a, b) => a.sequence - b.sequence);
    while (this.pendingOutOfOrder.length > 0 && this.pendingOutOfOrder[0].sequence === this.lastSequence + 1) {
      const next = this.pendingOutOfOrder.shift();
      this.events.push(next);
      this.lastSequence = next.sequence;
    }
  }

  requestBackfill(fromSeq, toSeq) {
    console.warn(`Gap detected: requesting events ${fromSeq}-${toSeq}`);
  }

  getScoringSummary() {
    const scoringTypes = {
      nfl: ['touchdown', 'field_goal', 'safety'],
      nba: ['basket', 'three_pointer', 'free_throw', 'dunk'],
      epl: ['goal'],
      mlb: ['run_scored', 'home_run']
    };
    return this.events.filter(e => (scoringTypes[e.sport] || []).includes(e.type));
  }
}
```

## Period and Clock Tracking

### Period Transition Map

| Sport | Periods | Labels | Clock Direction |
|-------|---------|--------|-----------------|
| NFL | 4 quarters + OT | Q1-Q4, OT | Counts down from 15:00 |
| NBA | 4 quarters + OT | Q1-Q4, OT | Counts down from 12:00 |
| NHL | 3 periods + OT | P1-P3, OT | Counts down from 20:00 |
| Soccer | 2 halves + ET | 1st Half, 2nd Half, ET1, ET2 | Counts up from 0:00 |
| MLB | 9 innings | Top/Bot 1st-9th, Extras | No clock |

### Period Label Formatting

```javascript
function formatPeriodLabel(sport, period, isTop) {
  switch (sport) {
    case 'nfl':
    case 'nba':
      return period <= 4 ? `Q${period}` : `OT${period - 4 > 1 ? period - 4 : ''}`;
    case 'nhl':
      return period <= 3 ? `P${period}` : `OT`;
    case 'epl':
      return period === 1 ? '1st Half' : period === 2 ? '2nd Half' : `ET${period - 2}`;
    case 'mlb': {
      const suffix = getOrdinalSuffix(period);
      return `${isTop ? 'Top' : 'Bot'} ${period}${suffix}`;
    }
    default: return `Period ${period}`;
  }
}

function getOrdinalSuffix(n) {
  if (n >= 11 && n <= 13) return 'th';
  switch (n % 10) { case 1: return 'st'; case 2: return 'nd'; case 3: return 'rd'; default: return 'th'; }
}
```

## Error Handling

### Out-of-Order Event Recovery

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
9. **Period labels** - Use sport-appropriate labels (quarters, halves, innings) in every event for correct rendering
10. **Scoring summary** - Maintain a separate list of scoring plays for quick catch-up when users join mid-game
