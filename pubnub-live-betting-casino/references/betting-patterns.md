# PubNub Betting & Casino Patterns

## Overview

This reference covers casino game state synchronization, in-play live betting patterns, responsible gambling features, regulatory compliance, multi-table tournaments, and social features for real-time betting and casino platforms.

## Casino Game State Synchronization

### Game Types and Architectures

| Game Type | State Model | Update Frequency | Channel Pattern |
|-----------|-------------|------------------|-----------------|
| Blackjack | Turn-based | Per action (hit, stand, deal) | `casino.blackjack.{tableId}` |
| Roulette | Round-based | Spin start, ball drop, result | `casino.roulette.{tableId}` |
| Slots | Instant | Spin result | `casino.slots.{userId}.{sessionId}` |
| Baccarat | Round-based | Deal, reveal, result | `casino.baccarat.{tableId}` |
| Poker | Turn-based | Per action, per street | `casino.poker.{tableId}` |
| Crash | Progressive | Continuous multiplier updates | `casino.crash.{roundId}` |

### Blackjack Table State

```javascript
async function publishBlackjackState(pubnub, tableId, gameState) {
  await pubnub.publish({
    channel: `casino.blackjack.${tableId}`,
    message: {
      type: 'game_state',
      tableId,
      phase: gameState.phase,  // 'betting', 'dealing', 'player_turn', 'dealer_turn', 'settlement'
      dealer: { cards: gameState.dealer.cards, total: gameState.dealer.total, faceDown: gameState.phase === 'player_turn' },
      players: gameState.players.map(p => ({
        userId: p.userId, seat: p.seat, cards: p.cards, total: p.total, bet: p.bet,
        status: p.status,  // 'active', 'stand', 'bust', 'blackjack'
        canHit: p.total < 21 && gameState.phase === 'player_turn',
        canDouble: p.cards.length === 2 && p.total >= 9 && p.total <= 11
      })),
      timestamp: Date.now()
    }
  });
}
```

### Blackjack PubNub Function (Game Logic)

```javascript
// PubNub Function: On Publish to 'casino.blackjack.*.action'
const db = require('kvstore');
const pubnub = require('pubnub');

export default async (request) => {
  const action = request.message;
  const gameState = await db.get(`blackjack:${action.tableId}`);
  if (!gameState) return request.ok();

  const player = gameState.players.find(p => p.userId === action.userId);
  if (!player || player.status !== 'active') return request.ok();

  switch (action.action) {
    case 'hit':
      player.cards.push(drawCard(gameState.deck));
      player.total = calculateTotal(player.cards);
      if (player.total > 21) player.status = 'bust';
      break;
    case 'stand': player.status = 'stand'; break;
    case 'double':
      if (player.cards.length !== 2) break;
      player.bet *= 2;
      player.cards.push(drawCard(gameState.deck));
      player.total = calculateTotal(player.cards);
      player.status = player.total > 21 ? 'bust' : 'stand';
      break;
  }

  await db.set(`blackjack:${action.tableId}`, gameState);
  await pubnub.publish({ channel: `casino.blackjack.${action.tableId}`, message: { type: 'game_state', ...gameState, timestamp: Date.now() } });
  return request.ok();
};
```

### Roulette Spin Sequence

```javascript
async function runRouletteRound(pubnub, tableId) {
  const channel = `casino.roulette.${tableId}`;
  const roundId = generateRoundId();

  await pubnub.publish({ channel, message: { type: 'phase_change', phase: 'betting_open', countdown: 30, roundId, timestamp: Date.now() } });
  await wait(25000);
  await pubnub.publish({ channel, message: { type: 'phase_change', phase: 'last_bets', countdown: 5, timestamp: Date.now() } });
  await wait(5000);
  await pubnub.publish({ channel, message: { type: 'phase_change', phase: 'no_more_bets', timestamp: Date.now() } });

  const result = generateRouletteResult();
  await pubnub.publish({ channel, message: { type: 'spin_result', result, winningBets: calculateWinners(result), timestamp: Date.now() } });
  await wait(5000);
  await settleRouletteBets(pubnub, tableId, result);
}
```

## In-Play / Live Betting

### Live Event State

```javascript
async function publishEventState(pubnub, eventId, state) {
  await pubnub.publish({
    channel: `event.football.${eventId}`,
    message: {
      type: 'event_state', eventId,
      status: state.status,  // 'first_half', 'half_time', 'second_half', 'full_time'
      clock: state.clock, homeScore: state.homeScore, awayScore: state.awayScore,
      incidents: state.recentIncidents,
      stats: { possession: state.possession, shots: state.shots, corners: state.corners },
      timestamp: Date.now()
    }
  });
}
```

### Incident-Triggered Market Suspension

```javascript
async function handleIncident(pubnub, eventId, incident) {
  const markets = await getEventMarkets(eventId);

  // Step 1: Suspend all markets immediately
  await Promise.all(markets.map(m =>
    pubnub.publish({ channel: `event.football.${eventId}.market.${m.id}`, message: { type: 'market_suspension', marketId: m.id, suspended: true, reason: incident.type, timestamp: Date.now() } })
  ));

  // Step 2: Publish incident
  await pubnub.publish({ channel: `event.football.${eventId}`, message: { type: 'incident', incident, timestamp: Date.now() } });

  // Step 3: Recalculate and reopen
  const newOdds = await recalculateOdds(eventId, incident);
  for (const m of markets) {
    if (m.shouldReopen) {
      await pubnub.publish({ channel: `event.football.${eventId}.market.${m.id}`, message: { type: 'odds_update', marketId: m.id, selections: newOdds[m.id], suspended: false, timestamp: Date.now() } });
    }
  }
}
```

## Responsible Gambling Features

### Deposit and Loss Limits

```javascript
const db = require('kvstore');

async function checkDepositLimit(userId, amount) {
  const limits = await db.get(`limits:${userId}`);
  if (!limits) return { allowed: true };

  const deposited = await db.get(`deposits:${userId}:${getCurrentPeriod(limits.period)}`) || { total: 0 };
  if (deposited.total + amount > limits.depositLimit) {
    return { allowed: false, reason: 'Deposit limit reached', limit: limits.depositLimit, resetsAt: getResetTime(limits.period) };
  }
  return { allowed: true };
}
```

### Session Tracking

```javascript
class SessionTracker {
  constructor(pubnub, userId) {
    this.pubnub = pubnub;
    this.userId = userId;
    this.sessionStart = Date.now();
    this.totalStaked = 0;
    this.totalWon = 0;
  }

  recordBet(stake) { this.totalStaked += stake; this.checkLimits(); }
  recordWin(payout) { this.totalWon += payout; }
  get netLoss() { return this.totalStaked - this.totalWon; }

  checkLimits() {
    if ((Date.now() - this.sessionStart) > 3600000) {
      this.alert('session_duration', 'You have been playing for over 1 hour');
    }
    if (this.netLoss > 100) {
      this.alert('loss_limit', `Net losses: $${this.netLoss.toFixed(2)}`);
    }
  }

  async alert(type, message) {
    await this.pubnub.publish({
      channel: `responsible.${this.userId}`,
      message: { type: 'responsible_gambling_alert', alertType: type, message, timestamp: Date.now() }
    });
  }
}
```

### Self-Exclusion

```javascript
// PubNub Function: Check self-exclusion before granting access
const db = require('kvstore');

export default async (request) => {
  const exclusion = await db.get(`exclusion:${request.message.userId}`);
  if (exclusion && exclusion.active && Date.now() < exclusion.expiresAt) {
    request.message = { type: 'access_denied', reason: 'self_excluded', expiresAt: exclusion.expiresAt };
    return request.abort();
  }
  return request.ok();
};
```

## Regulatory Compliance

### Geo-Fencing

```javascript
async function verifyGeolocation(pubnub, userId) {
  return new Promise((resolve, reject) => {
    navigator.geolocation.getCurrentPosition(
      async (pos) => {
        await pubnub.publish({ channel: 'geo.verify', message: { userId, latitude: pos.coords.latitude, longitude: pos.coords.longitude, timestamp: Date.now() } });
        resolve(true);
      },
      (err) => reject(new Error(`Location denied: ${err.message}`)),
      { enableHighAccuracy: true, timeout: 10000 }
    );
  });
}
```

### Geo-Fence PubNub Function

```javascript
const db = require('kvstore');
const xhr = require('xhr');

export default async (request) => {
  const { userId, latitude, longitude } = request.message;
  const response = await xhr.fetch(`https://api.geocoding-service.com/reverse?lat=${latitude}&lon=${longitude}`);
  const location = JSON.parse(response.body);
  const ALLOWED = ['GB', 'MT', 'GI', 'IE', 'SE', 'DK'];

  if (!ALLOWED.includes(location.countryCode)) {
    await db.set(`geo:${userId}`, { allowed: false, country: location.countryCode });
    return request.abort();
  }

  await db.set(`geo:${userId}`, { allowed: true, country: location.countryCode, verifiedAt: Date.now(), expiresAt: Date.now() + 1800000 });
  return request.ok();
};
```

## Multi-Table Tournament Support

### Tournament Channel Structure

| Channel Pattern | Purpose |
|----------------|---------|
| `tournament.{id}` | Tournament-wide announcements |
| `tournament.{id}.table.{tableId}` | Individual table game state |
| `tournament.{id}.leaderboard` | Live leaderboard updates |
| `tournament.{id}.lobby` | Pre-tournament lobby chat |

### Tournament Lobby and Table Assignment

```javascript
async function joinTournament(pubnub, tournamentId, userId) {
  pubnub.subscribe({ channels: [`tournament.${tournamentId}`, `tournament.${tournamentId}.lobby`, `tournament.${tournamentId}.leaderboard`] });
  await pubnub.publish({ channel: `tournament.${tournamentId}.lobby`, message: { type: 'player_joined', userId, timestamp: Date.now() } });
}

async function startTournament(pubnub, tournament) {
  const tables = assignPlayersToTables(tournament.players, tournament.playersPerTable);
  for (const table of tables) {
    await pubnub.publish({ channel: `tournament.${tournament.id}`, message: { type: 'table_assignment', tableId: table.id, players: table.players, timestamp: Date.now() } });
  }
}
```

### Leaderboard Updates

```javascript
async function updateLeaderboard(pubnub, tournamentId, standings) {
  await pubnub.publish({
    channel: `tournament.${tournamentId}.leaderboard`,
    message: { type: 'leaderboard_update', standings: standings.map((p, i) => ({ rank: i + 1, userId: p.userId, displayName: p.displayName, chips: p.chips, status: p.status })), timestamp: Date.now() }
  });
}
```

## Social Features

### Bet Sharing

```javascript
async function shareBet(pubnub, userId, bet) {
  await pubnub.publish({
    channel: 'social.feed',
    message: { type: 'shared_bet', sharedBy: userId, betId: bet.betId, selections: bet.selections.map(s => ({ selectionName: s.selectionName, odds: s.oddsAtSelection })), stake: bet.stake, potentialReturn: bet.potentialReturn, timestamp: Date.now() }
  });
}
```

## Swift Casino Client (iOS)

```swift
import PubNub

class RouletteViewController: UIViewController {
    var pubnub: PubNub!

    override func viewDidLoad() {
        super.viewDidLoad()
        let config = PubNubConfiguration(publishKey: "pub-c-...", subscribeKey: "sub-c-...", userId: "ios-player-123")
        pubnub = PubNub(configuration: config)
        pubnub.subscribe(to: ["casino.roulette.table-01"])

        let listener = SubscriptionListener()
        listener.didReceiveMessage = { [weak self] message in
            guard let payload = message.payload.dictionaryOptional,
                  let type = payload["type"] as? String else { return }
            DispatchQueue.main.async {
                switch type {
                case "phase_change": self?.handlePhaseChange(payload)
                case "spin_result": self?.handleSpinResult(payload)
                default: break
                }
            }
        }
        pubnub.add(listener)
    }
}
```

## Kotlin Casino Client (Android)

```kotlin
class BlackjackActivity : AppCompatActivity() {
    private lateinit var pubnub: PubNub

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val config = PNConfiguration(userId = UserId("android-player-456")).apply {
            subscribeKey = "sub-c-..."; publishKey = "pub-c-..."; cipherKey = "encryption-key"
        }
        pubnub = PubNub.create(config)
        pubnub.subscribe(channels = listOf("casino.blackjack.table-01"))

        pubnub.addListener(object : SubscribeCallback() {
            override fun message(pubnub: PubNub, result: PNMessageResult) {
                val type = result.message.asJsonObject.get("type").asString
                runOnUiThread { when (type) {
                    "game_state" -> updateGameUI(result.message.asJsonObject)
                    "phase_change" -> handlePhaseChange(result.message.asJsonObject)
                } }
            }
        })
    }
}
```

## Error Handling

```javascript
pubnub.addListener({
  status: (statusEvent) => {
    if (statusEvent.category === 'PNReconnectedCategory') {
      fetchCurrentGameState(currentTableId).then(state => updateGameUI(state));
    }
    if (statusEvent.category === 'PNNetworkIssuesCategory') {
      showOverlay('Connection lost. Reconnecting...');
      disableBettingControls();
    }
  }
});
```

## Best Practices

1. **Use dedicated channels per game table** to isolate game state and prevent cross-table interference
2. **Run all game logic in PubNub Functions** to ensure the server is the single source of truth
3. **Include sequence numbers** in game state messages to detect out-of-order delivery
4. **Suspend markets before recalculation** to prevent bets on stale odds during live events
5. **Enforce responsible gambling limits server-side** using PubNub Functions
6. **Re-verify geolocation periodically** (every 30 minutes) to detect VPN usage
7. **Use PubNub Presence** to track active players and manage seat availability
8. **Encrypt all financial channels** (wagers, balances, cash-out) with AES-256
9. **Design tournament channels hierarchically** so players subscribe only to relevant feeds
10. **Rate-limit social features** (chat, bet sharing) to prevent spam
11. **Log all game state transitions** to Message Persistence for dispute resolution
12. **Implement cool-down periods** between rapid betting actions to promote responsible gambling
