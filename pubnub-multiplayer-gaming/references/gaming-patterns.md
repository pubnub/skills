# PubNub Multiplayer Gaming Patterns

## Overview

PubNub publish/subscribe wiring for matchmaking, turn-based rooms, anti-cheat validation, leaderboards, spectator feeds, and in-game chat.

## Matchmaking Implementation

Queue on `matchmaking.{gameType}.*`; notify pairs on `player.{playerId}`. Keep pairing logic in a PubNub Function so clients cannot forge queue position.

### Simple Random Matchmaking

```javascript
class RandomMatchmaker {
  constructor(pubnub, gameType) {
    this.pubnub = pubnub;
    this.gameType = gameType;
    this.queueChannel = `matchmaking.${gameType}.random`;
    this.playerChannel = `player.${pubnub.getUserId()}`;
  }

  async joinQueue() {
    this.pubnub.subscribe({
      channels: [this.playerChannel, this.queueChannel]
    });

    await this.pubnub.publish({
      channel: this.queueChannel,
      message: {
        type: 'queue-join',
        playerId: this.pubnub.getUserId(),
        timestamp: Date.now()
      }
    });
  }

  setupListener(onMatchFound) {
    this.pubnub.addListener({
      message: (event) => {
        if (event.channel === this.playerChannel &&
            event.message.type === 'match-found') {
          onMatchFound(event.message);
        }
      }
    });
  }

  async leaveQueue() {
    await this.pubnub.publish({
      channel: this.queueChannel,
      message: {
        type: 'queue-leave',
        playerId: this.pubnub.getUserId()
      }
    });
    this.pubnub.unsubscribe({ channels: [this.queueChannel] });
  }
}
```

### Skill-Based Matchmaking with PubNub Functions

```javascript
// PubNub Function: On Request handler for matchmaking
// Endpoint: POST /matchmake
export default async (request, response) => {
  const db = require('kvstore');
  const body = JSON.parse(request.body);
  const { playerId, rating, gameType } = body;

  const queueKey = `queue_${gameType}`;
  let queue = (await db.get(queueKey)) || [];

    const RATING_RANGE = 200;
    const match = queue.find(
      (p) => Math.abs(p.rating - rating) <= RATING_RANGE
    );

    if (match) {
      queue = queue.filter((p) => p.playerId !== match.playerId);
      db.set(queueKey, queue);

      const roomId = `${Date.now()}-${Math.random().toString(36).slice(2, 6)}`;
      const pubnub = require('pubnub');

      pubnub.publish({
        channel: `player.${playerId}`,
        message: {
          type: 'match-found',
          roomId,
          opponent: match.playerId,
          opponentRating: match.rating
        }
      });

      pubnub.publish({
        channel: `player.${match.playerId}`,
        message: {
          type: 'match-found',
          roomId,
          opponent: playerId,
          opponentRating: rating
        }
      });

      response.status = 200;
      return response.send({ matched: true, roomId });
    } else {
      queue.push({ playerId, rating, joinedAt: Date.now() });
      db.set(queueKey, queue);

      response.status = 200;
      return response.send({ matched: false, position: queue.length });
    }
};
```

## Turn-Based Game Patterns

Publish turn events on `game.{roomId}.state`. Enforce whose turn it is in a Function (see Anti-Cheat) rather than trusting the client.

```javascript
class TurnManager {
  constructor(pubnub, roomId, players) {
    this.pubnub = pubnub;
    this.roomId = roomId;
    this.stateChannel = `game.${roomId}.state`;
    this.players = players;
    this.currentTurnIndex = 0;
    this.turnNumber = 0;
    this.turnTimeout = 30000;
    this.turnTimer = null;
  }

  get currentPlayer() {
    return this.players[this.currentTurnIndex];
  }

  async startTurn() {
    this.turnNumber++;

    await this.pubnub.publish({
      channel: this.stateChannel,
      message: {
        type: 'turn-start',
        turnNumber: this.turnNumber,
        activePlayer: this.currentPlayer,
        deadline: Date.now() + this.turnTimeout,
        timestamp: Date.now()
      }
    });

    this.turnTimer = setTimeout(() => {
      this.handleTurnTimeout();
    }, this.turnTimeout);
  }

  async submitAction(playerId, action) {
    if (playerId !== this.currentPlayer) {
      throw new Error('Not your turn');
    }

    clearTimeout(this.turnTimer);

    await this.pubnub.publish({
      channel: this.stateChannel,
      message: {
        type: 'turn-action',
        turnNumber: this.turnNumber,
        playerId,
        action,
        timestamp: Date.now()
      }
    });

    this.currentTurnIndex = (this.currentTurnIndex + 1) % this.players.length;
    await this.startTurn();
  }

  async handleTurnTimeout() {
    await this.pubnub.publish({
      channel: this.stateChannel,
      message: {
        type: 'turn-timeout',
        turnNumber: this.turnNumber,
        timedOutPlayer: this.currentPlayer,
        timestamp: Date.now()
      }
    });

    this.currentTurnIndex = (this.currentTurnIndex + 1) % this.players.length;
    await this.startTurn();
  }

  destroy() {
    if (this.turnTimer) {
      clearTimeout(this.turnTimer);
    }
  }
}
```

## Anti-Cheat Validation via PubNub Functions

Use [Pattern 1 Before Publish](../../pubnub-functions/references/functions-patterns.md#pattern-1-distributed-counter) on `game.*.state` channels. **Game-specific checks only:**

| Input | Rule |
|-------|------|
| `player-input` move | Max speed / distance per tick |
| `state-delta` health | Cap decrement magnitude |
| Timestamp | Reject if >5s in the future |
| On fail | `request.abort()` or replace with `action-rejected` |

Full deployable handler: start from the functions owner; apply rules from [gaming-state-sync.md](gaming-state-sync.md).

### Server-Side Score Validation

```javascript
// PubNub Function: On Request handler for score submission
// Endpoint: POST /submit-score
export default async (request, response) => {
  const db = require('kvstore');
  const body = JSON.parse(request.body);
  const { playerId, roomId, score, gameEvents } = body;

  let calculatedScore = 0;
  for (const event of gameEvents) {
    switch (event.type) {
      case 'kill':
        calculatedScore += 100;
        break;
      case 'assist':
        calculatedScore += 50;
        break;
      case 'objective':
        calculatedScore += 200;
        break;
    }
  }

  if (Math.abs(calculatedScore - score) > 10) {
    response.status = 403;
    return response.send({ error: 'Score mismatch detected' });
  }

  let leaderboard = (await db.get('leaderboard')) || [];
  leaderboard.push({
    playerId,
    score: calculatedScore,
    roomId,
    timestamp: Date.now()
  });

  leaderboard.sort((a, b) => b.score - a.score);
  leaderboard = leaderboard.slice(0, 100);

  await db.set('leaderboard', leaderboard);
  response.status = 200;
  return response.send({ verified: true, score: calculatedScore });
};
```

## Leaderboard Management

### Real-Time Leaderboard with PubNub

```javascript
class GameLeaderboard {
  constructor(pubnub, gameType) {
    this.pubnub = pubnub;
    this.gameType = gameType;
    this.leaderboardChannel = `leaderboard.${gameType}`;
    this.entries = [];
  }

  async fetchLeaderboard() {
    const result = await fetch(
      `https://ps.pndsn.com/v1/blocks/sub-key/sub-c-.../leaderboard?type=${this.gameType}`
    );
    this.entries = await result.json();
    return this.entries;
  }

  subscribeToUpdates(onUpdate) {
    this.pubnub.subscribe({ channels: [this.leaderboardChannel] });

    this.pubnub.addListener({
      message: (event) => {
        if (event.channel === this.leaderboardChannel) {
          const msg = event.message;

          if (msg.type === 'score-update') {
            this.updateEntry(msg.playerId, msg.score);
            onUpdate(this.entries);
          }
        }
      }
    });
  }

  updateEntry(playerId, score) {
    const existing = this.entries.find((e) => e.playerId === playerId);
    if (existing) {
      existing.score = Math.max(existing.score, score);
    } else {
      this.entries.push({ playerId, score, timestamp: Date.now() });
    }
    this.entries.sort((a, b) => b.score - a.score);
    this.entries = this.entries.slice(0, 100);
  }

  cleanup() {
    this.pubnub.unsubscribe({ channels: [this.leaderboardChannel] });
  }
}
```

## In-Game Chat Integration

Keep chat on `game.{roomId}.chat` so it never shares a channel with `state` traffic.

```javascript
class InGameChat {
  constructor(pubnub, roomId) {
    this.pubnub = pubnub;
    this.chatChannel = `game.${roomId}.chat`;
  }

  connect(onMessage) {
    this.pubnub.subscribe({ channels: [this.chatChannel] });

    this.pubnub.addListener({
      message: (event) => {
        if (event.channel === this.chatChannel) {
          onMessage({
            senderId: event.message.senderId,
            text: event.message.text,
            timestamp: event.timetoken,
            team: event.message.team || null
          });
        }
      }
    });
  }

  async sendMessage(text, team = null) {
    await this.pubnub.publish({
      channel: this.chatChannel,
      message: {
        type: 'chat',
        senderId: this.pubnub.getUserId(),
        text,
        team
      }
    });
  }

  disconnect() {
    this.pubnub.unsubscribe({ channels: [this.chatChannel] });
  }
}
```

## Spectator Mode Implementation

### Spectator Channel Pattern

Spectators subscribe to `game.{roomId}.spectator` (read-only). Hosts optionally delay republish from `.state` to reduce stream sniping. Use `hereNow` on the spectator channel for live viewer counts.

```javascript
class SpectatorManager {
  constructor(pubnub, roomId) {
    this.pubnub = pubnub;
    this.roomId = roomId;
    this.spectatorChannel = `game.${roomId}.spectator`;
    this.stateChannel = `game.${roomId}.state`;
  }

  async joinAsSpectator() {
    this.pubnub.subscribe({
      channels: [this.spectatorChannel]
    });

    await this.pubnub.publish({
      channel: this.stateChannel,
      message: {
        type: 'spectator-join',
        spectatorId: this.pubnub.getUserId()
      }
    });
  }

  broadcastToSpectators(gameState, delay = 3000) {
    setTimeout(() => {
      this.pubnub.publish({
        channel: this.spectatorChannel,
        message: {
          type: 'spectator-update',
          state: gameState,
          timestamp: Date.now()
        }
      });
    }, delay);
  }

  async getSpectatorCount() {
    const result = await this.pubnub.hereNow({
      channels: [this.spectatorChannel],
      includeUUIDs: false
    });
    return result.channels[this.spectatorChannel]?.occupancy || 0;
  }

  leave() {
    this.pubnub.unsubscribe({ channels: [this.spectatorChannel] });
  }
}
```

## Best Practices

1. **Keep matchmaking server-side** -- use PubNub Functions (On Request or After Publish) for matchmaking logic to prevent clients from manipulating queue position or pairing.

2. **Use a spectator delay** -- broadcast game state to spectators with a 3-5 second delay to prevent stream sniping in competitive games.

3. **Validate game actions on the server** -- use PubNub Functions Before Publish triggers to reject impossible actions before they reach other players.

4. **Store leaderboards in PubNub KV Store** -- persist via Functions KV; broadcast updates on a leaderboard channel.

5. **Separate chat from game state channels** -- game state messages should never be delayed by chat traffic.

6. **Use presence for player counts in lobbies** -- call `hereNow` to show live player counts in room listings rather than maintaining your own counters.

7. **Rate-limit client publishes** -- enforce a maximum message rate on the client side; PubNub Functions can add server-side rate limiting as well.

8. **Keep mobile battery usage in mind** -- use longer heartbeat intervals on mobile when the game is in the background and reduce update frequency for non-critical data.
