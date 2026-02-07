# PubNub Sport Updates Patterns

## Multi-Sport Dashboard

### Dashboard Architecture

A multi-sport dashboard subscribes to multiple league channels simultaneously and renders a unified view of all active games.

```javascript
class MultiSportDashboard {
  constructor(pubnub) {
    this.pubnub = pubnub;
    this.games = new Map();
    this.listeners = new Set();

    this.pubnub.addListener({
      message: (event) => this.handleMessage(event),
      status: (event) => this.handleStatus(event)
    });
  }

  subscribeToLeagues(leagues) {
    this.leagues = leagues;
    const channels = leagues.flatMap(league => [
      `sports.${league}.scores`,
      `sports.${league}.games.*`
    ]);
    this.pubnub.subscribe({ channels });
  }

  handleMessage(event) {
    const { message } = event;
    if (message.type === 'score_update' || message.type === 'score_summary') {
      this.games.set(message.gameId, { ...this.games.get(message.gameId), ...message, lastUpdated: Date.now() });
    }
    if (message.type === 'game_status' && message.payload?.status === 'final') {
      setTimeout(() => { this.games.delete(message.gameId); this.notifyListeners(); }, 300000);
    }
    this.notifyListeners();
  }

  handleStatus(event) {
    if (event.category === 'PNReconnectedCategory') this.refreshAllGames();
  }

  getGamesByLeague(league) {
    return [...this.games.values()]
      .filter(g => g.sport === league)
      .sort((a, b) => {
        const order = { in_progress: 0, halftime: 1, pre_game: 2, final: 3 };
        return (order[a.status] || 9) - (order[b.status] || 9);
      });
  }

  onUpdate(callback) {
    this.listeners.add(callback);
    return () => this.listeners.delete(callback);
  }

  notifyListeners() {
    for (const cb of this.listeners) cb(this.games);
  }

  async refreshAllGames() {
    for (const league of this.leagues) {
      try {
        const response = await this.pubnub.fetchMessages({ channels: [`sports.${league}.scores`], count: 25 });
        for (const entry of (response.channels[`sports.${league}.scores`] || [])) {
          this.games.set(entry.message.gameId, { ...this.games.get(entry.message.gameId), ...entry.message });
        }
      } catch (error) { console.error(`Failed to refresh ${league}:`, error); }
    }
    this.notifyListeners();
  }

  destroy() {
    this.pubnub.unsubscribeAll();
    this.listeners.clear();
    this.games.clear();
  }
}
```

### React Dashboard Component

```javascript
import { useState, useEffect } from 'react';

function SportsDashboard({ pubnub, leagues }) {
  const [dashboard] = useState(() => new MultiSportDashboard(pubnub));
  const [games, setGames] = useState(new Map());

  useEffect(() => {
    dashboard.subscribeToLeagues(leagues);
    const unsub = dashboard.onUpdate((g) => setGames(new Map(g)));
    return () => { unsub(); dashboard.destroy(); };
  }, [dashboard, leagues]);

  return (
    <div className="dashboard">
      {leagues.map(league => (
        <LeagueSection key={league} league={league} games={dashboard.getGamesByLeague(league)} />
      ))}
    </div>
  );
}

function ScoreCard({ game }) {
  return (
    <div className={`score-card status-${game.status}`}>
      <div className="team away">
        <span className="name">{game.away?.name}</span>
        <span className="score">{game.away?.score}</span>
      </div>
      <div className="team home">
        <span className="name">{game.home?.name}</span>
        <span className="score">{game.home?.score}</span>
      </div>
      <div className="game-info">
        <span className="period">{game.period?.label}</span>
        {game.period?.clock && <span className="clock">{game.period.clock}</span>}
      </div>
    </div>
  );
}
```

## Fan Engagement Features

### Live Polls

```javascript
class GamePoll {
  constructor(pubnub, gameId) {
    this.pubnub = pubnub;
    this.channel = `sports.fan.polls.${gameId}`;
  }

  async createPoll(poll) {
    await this.pubnub.publish({
      channel: this.channel,
      message: { type: 'poll_created', pollId: poll.id, question: poll.question, options: poll.options, expiresAt: Date.now() + poll.durationMs }
    });
  }

  async castVote(pollId, optionIndex, userId) {
    await this.pubnub.publish({
      channel: this.channel,
      message: { type: 'poll_vote', pollId, optionIndex, userId, timestamp: Date.now() }
    });
  }

  subscribe(onPollEvent) {
    const listener = { message: (event) => { if (event.channel === this.channel) onPollEvent(event.message); } };
    this.pubnub.addListener(listener);
    this.pubnub.subscribe({ channels: [this.channel] });
    return () => { this.pubnub.removeListener(listener); this.pubnub.unsubscribe({ channels: [this.channel] }); };
  }
}
```

### Live Reactions

```javascript
class GameReactions {
  constructor(pubnub, gameId) {
    this.pubnub = pubnub;
    this.channel = `sports.fan.reactions.${gameId}`;
    this.counts = {};
  }

  async sendReaction(emoji, userId) {
    await this.pubnub.publish({
      channel: this.channel,
      message: { type: 'reaction', emoji, userId, timestamp: Date.now() }
    });
  }

  subscribe(onReaction) {
    const listener = {
      message: (event) => {
        if (event.channel === this.channel && event.message.type === 'reaction') {
          const { emoji } = event.message;
          this.counts[emoji] = (this.counts[emoji] || 0) + 1;
          onReaction(event.message, { ...this.counts });
        }
      }
    };
    this.pubnub.addListener(listener);
    this.pubnub.subscribe({ channels: [this.channel] });
    return () => { this.pubnub.removeListener(listener); this.pubnub.unsubscribe({ channels: [this.channel] }); };
  }
}
```

### Live Predictions

```javascript
async function submitPrediction(pubnub, gameId, userId, prediction) {
  await pubnub.publish({
    channel: `sports.fan.predictions.${gameId}`,
    message: {
      type: 'prediction',
      gameId, userId,
      prediction: { winner: prediction.winner, finalScore: prediction.finalScore, mvp: prediction.mvp },
      timestamp: Date.now()
    }
  });
}
```

## Personalization

### Favorite Teams and Custom Alerts

```javascript
class FanPreferences {
  constructor(pubnub, userId) {
    this.pubnub = pubnub;
    this.userId = userId;
    this.favorites = [];
  }

  async loadPreferences() {
    try {
      const response = await this.pubnub.objects.getUUIDMetadata({ uuid: this.userId, include: { customFields: true } });
      this.favorites = response.data.custom?.favoriteTeams || [];
    } catch (error) {
      console.error('Failed to load preferences:', error);
      this.favorites = [];
    }
  }

  async savePreferences(teams) {
    this.favorites = teams;
    await this.pubnub.objects.setUUIDMetadata({
      uuid: this.userId,
      data: { custom: { favoriteTeams: teams } }
    });
  }

  subscribeToFavorites() {
    const channels = this.favorites.flatMap(fav => [
      `sports.${fav.league}.teams.${fav.teamId}`,
      `sports.${fav.league}.teams.${fav.teamId}.*`
    ]);
    this.pubnub.subscribe({ channels });
    return channels;
  }
}

// Usage
const prefs = new FanPreferences(pubnub, 'fan-user-123');
await prefs.loadPreferences();
await prefs.savePreferences([
  { league: 'nfl', teamId: 'SF' },
  { league: 'nba', teamId: 'GSW' },
  { league: 'epl', teamId: 'ARS' }
]);
prefs.subscribeToFavorites();
```

## Push Notifications for Key Events

### Mobile Push Configuration

```javascript
async function registerForPush(pubnub, deviceToken, platform, favoriteTeams) {
  const channels = favoriteTeams.map(fav => `sports.${fav.league}.teams.${fav.teamId}`);

  if (platform === 'ios') {
    await pubnub.push.addChannels({
      channels, device: deviceToken, pushGateway: 'apns2',
      environment: 'production', topic: 'com.example.sportsapp'
    });
  } else {
    await pubnub.push.addChannels({ channels, device: deviceToken, pushGateway: 'gcm' });
  }
}
```

### Push Notification Payload Construction

```javascript
function buildPushPayload(event) {
  const title = `${event.score.away.team} ${event.score.away.score} - ${event.score.home.team} ${event.score.home.score}`;
  const body = getPushBody(event);

  return {
    pn_apns: {
      aps: { alert: { title, body }, sound: 'score_update.aiff', badge: 1 },
      gameId: event.gameId, type: event.type
    },
    pn_gcm: {
      notification: { title, body, icon: 'ic_score', sound: 'default' },
      data: { gameId: event.gameId, type: event.type }
    }
  };
}

function getPushBody(event) {
  switch (event.type) {
    case 'goal': return `GOAL! ${event.payload.scorer} scores for ${event.payload.team}`;
    case 'touchdown': return `TOUCHDOWN! ${event.payload.player} - ${event.payload.description}`;
    case 'home_run': return `HOME RUN! ${event.payload.batter} - ${event.payload.description}`;
    case 'game_status':
      if (event.payload.status === 'final') return 'FINAL';
      if (event.payload.status === 'in_progress') return 'Game has started!';
      return event.payload.statusMessage;
    default: return event.payload?.description || '';
  }
}

async function publishWithPush(pubnub, channel, event) {
  await pubnub.publish({ channel, message: { ...event, ...buildPushPayload(event) } });
}
```

## Scaling for Major Events

### Delivery Strategy Comparison

| Strategy | Throughput | Latency | Use Case |
|----------|-----------|---------|----------|
| Per-game channels | High | Low (<100ms) | Default for all games |
| Wildcard subscription | High | Low | Multi-game dashboards |
| League score channel | Very High | Low | Score tickers, league overviews |
| PubNub Functions filter | Medium | Medium (+50ms) | Personalized filtering |
| Delta compression | Very High | Low | High-frequency stat updates |

### PubNub Functions for Event Enrichment

```javascript
// PubNub Function: Before Publish handler
export default (request) => {
  const message = request.message;

  if (message.type === 'score_update') {
    message.computed = {
      scoreDifferential: Math.abs(message.home.score - message.away.score),
      isCloseGame: Math.abs(message.home.score - message.away.score) <= 7,
      leader: message.home.score > message.away.score ? message.home.team
        : message.home.score < message.away.score ? message.away.team : 'tied'
    };
  }

  return request.ok();
};
```

### Delta Compression for High-Frequency Updates

```javascript
class DeltaCompressor {
  constructor() { this.lastPublished = new Map(); }

  computeDelta(gameId, fullState) {
    const previous = this.lastPublished.get(gameId);
    if (!previous) { this.lastPublished.set(gameId, { ...fullState }); return { type: 'full_state', ...fullState }; }

    const delta = { type: 'delta', gameId, sequence: fullState.sequence, changes: {} };
    let hasChanges = false;
    for (const key of Object.keys(fullState)) {
      if (JSON.stringify(fullState[key]) !== JSON.stringify(previous[key])) {
        delta.changes[key] = fullState[key];
        hasChanges = true;
      }
    }
    if (hasChanges) { this.lastPublished.set(gameId, { ...fullState }); return delta; }
    return null;
  }
}

class DeltaApplier {
  constructor() { this.gameStates = new Map(); }

  apply(message) {
    if (message.type === 'full_state') { this.gameStates.set(message.gameId, message); return message; }
    if (message.type === 'delta') {
      const current = this.gameStates.get(message.gameId) || {};
      const updated = { ...current, ...message.changes, sequence: message.sequence };
      this.gameStates.set(message.gameId, updated);
      return updated;
    }
    return message;
  }
}
```

## Historical Data and Replay

### Game Replay from History

```javascript
async function replayGame(pubnub, league, gameId, onEvent, speedMultiplier = 10) {
  const channel = `sports.${league}.games.${gameId}.plays`;
  const response = await pubnub.fetchMessages({ channels: [channel], count: 100 });

  const events = (response.channels[channel] || [])
    .map(entry => entry.message)
    .sort((a, b) => a.sequence - b.sequence);

  let previousTimestamp = events[0]?.timestamp || Date.now();
  for (const event of events) {
    const delay = (event.timestamp - previousTimestamp) / speedMultiplier;
    if (delay > 0) await new Promise(resolve => setTimeout(resolve, Math.min(delay, 5000)));
    onEvent(event);
    previousTimestamp = event.timestamp;
  }
}
```

### Game Summary Construction

```javascript
async function buildGameSummary(pubnub, league, gameId) {
  const channel = `sports.${league}.games.${gameId}`;
  const response = await pubnub.fetchMessages({ channels: [channel], count: 100 });
  const messages = (response.channels[channel] || []).map(e => e.message);

  const finalScore = messages.filter(m => m.type === 'score_update').sort((a, b) => b.sequence - a.sequence)[0];
  const scoringPlays = messages.filter(m =>
    ['touchdown', 'field_goal', 'goal', 'home_run', 'run_scored', 'three_pointer', 'dunk'].includes(m.type)
  );

  return {
    gameId, league,
    finalScore: finalScore ? { home: finalScore.home || finalScore.score?.home, away: finalScore.away || finalScore.score?.away } : null,
    scoringPlays: scoringPlays.map(p => ({ period: p.period.label, clock: p.period.clock, type: p.type, description: p.payload?.description }))
  };
}
```

## Error Handling Patterns

### Graceful Degradation

```javascript
class ResilientSportsClient {
  constructor(pubnub) {
    this.pubnub = pubnub;
    this.isConnected = false;
    this.reconnectAttempts = 0;

    this.pubnub.addListener({
      status: (event) => {
        switch (event.category) {
          case 'PNConnectedCategory':
            this.isConnected = true;
            this.reconnectAttempts = 0;
            break;
          case 'PNNetworkDownCategory':
            this.isConnected = false;
            break;
          case 'PNReconnectedCategory':
            this.isConnected = true;
            this.reconnectAttempts = 0;
            break;
          case 'PNNetworkIssuesCategory':
            this.reconnectAttempts++;
            if (this.reconnectAttempts >= 10) {
              console.warn('Unable to connect. Scores may be delayed.');
            }
            break;
        }
      }
    });
  }
}
```

## Best Practices

1. **Separate concerns by channel** - Use distinct channels for scores, play-by-play, fan engagement, and statistics so each client subscribes only to what it needs
2. **Wildcard subscriptions for dashboards** - Use `sports.<league>.games.*` for multi-game views rather than subscribing to dozens of individual channels
3. **Delta compression** - For high-frequency updates (pitch-by-pitch, possession tracking), send only changed fields to minimize bandwidth
4. **Push notification thresholds** - Send push notifications only for critical events (goals, game start, game end) to avoid notification fatigue
5. **Reconnection with backfill** - Always fetch recent history on reconnect; use sequence numbers to detect what was missed
6. **Graceful degradation** - Show cached scores with a staleness indicator when the connection is lost rather than an error state
7. **PubNub Functions enrichment** - Use before-publish Functions to compute derived fields server-side so all clients benefit without duplicating logic
8. **Cleanup subscriptions** - Always unsubscribe and destroy PubNub instances when components unmount or users navigate away
9. **Personalization via metadata** - Store favorite teams in PubNub App Context so preferences persist across sessions and devices
10. **Replay for late joiners** - Fetch the last N events from history and replay them through the timeline builder before subscribing to live updates
11. **Test at scale** - Simulate peak event traffic in staging to validate channel architecture and client rendering performance
