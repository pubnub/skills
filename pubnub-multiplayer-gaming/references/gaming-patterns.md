# PubNub Multiplayer Gaming Patterns

## Overview

This guide covers common multiplayer game patterns built on PubNub, including matchmaking, turn-based and real-time game architectures, anti-cheat validation, leaderboards, spectator mode, and in-game chat. Each pattern is designed to work within PubNub's publish/subscribe model.

## Matchmaking Implementation

### Matchmaking Approaches

| Approach | Description | Best For | Complexity |
|----------|-------------|----------|-----------|
| **Random** | First-come-first-served pairing | Casual games, quick play | Low |
| **Skill-Based (ELO/MMR)** | Match players within a rating range | Competitive games | Medium |
| **Ranked Tiers** | Group by rank brackets (Bronze, Silver, etc.) | Ranked modes | Medium |
| **Region-Based** | Match by geographic proximity | Latency-sensitive games | Medium |
| **Custom Criteria** | Match by level, loadout, preferences | RPGs, team games | High |

### Simple Random Matchmaking

```javascript
class RandomMatchmaker {
  constructor(pubnub, gameType) {
    this.pubnub = pubnub;
    this.gameType = gameType;
    this.queueChannel = `matchmaking.${gameType}.random`;
    this.playerChannel = `player.${pubnub.getUUID()}`;
  }

  async joinQueue() {
    // Subscribe to personal channel for match notifications
    this.pubnub.subscribe({
      channels: [this.playerChannel, this.queueChannel]
    });

    // Announce availability
    await this.pubnub.publish({
      channel: this.queueChannel,
      message: {
        type: 'queue-join',
        playerId: this.pubnub.getUUID(),
        timestamp: Date.now()
      }
    });
  }

  // A matchmaking service (PubNub Function) pairs players
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
        playerId: this.pubnub.getUUID()
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
export default (request, response) => {
  const db = require('kvstore');
  const body = JSON.parse(request.body);
  const { playerId, rating, gameType } = body;

  const queueKey = `queue_${gameType}`;

  return db.get(queueKey).then((queue) => {
    queue = queue || [];

    // Find a suitable opponent within rating range
    const RATING_RANGE = 200;
    const match = queue.find(
      (p) => Math.abs(p.rating - rating) <= RATING_RANGE
    );

    if (match) {
      // Remove matched player from queue
      queue = queue.filter((p) => p.playerId !== match.playerId);
      db.set(queueKey, queue);

      // Create a game room
      const roomId = `${Date.now()}-${Math.random().toString(36).slice(2, 6)}`;

      // Notify both players
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
      // Add to queue
      queue.push({ playerId, rating, joinedAt: Date.now() });
      db.set(queueKey, queue);

      response.status = 200;
      return response.send({ matched: false, position: queue.length });
    }
  });
};
```

### Expanding Search Range Over Time

```javascript
class AdaptiveMatchmaker {
  constructor(pubnub, playerId, rating) {
    this.pubnub = pubnub;
    this.playerId = playerId;
    this.rating = rating;
    this.searchRange = 100;
    this.maxRange = 500;
    this.expandInterval = 5000; // Expand every 5 seconds
    this.expandTimer = null;
  }

  async startSearching() {
    this.expandTimer = setInterval(() => {
      this.searchRange = Math.min(this.searchRange + 50, this.maxRange);
      this.refreshSearch();
    }, this.expandInterval);

    await this.refreshSearch();
  }

  async refreshSearch() {
    await this.pubnub.publish({
      channel: 'matchmaking.ranked',
      message: {
        type: 'search-update',
        playerId: this.playerId,
        rating: this.rating,
        range: this.searchRange,
        timestamp: Date.now()
      }
    });
  }

  stopSearching() {
    if (this.expandTimer) {
      clearInterval(this.expandTimer);
      this.expandTimer = null;
    }
  }
}
```

## Turn-Based Game Patterns

### Turn Manager

```javascript
class TurnManager {
  constructor(pubnub, roomId, players) {
    this.pubnub = pubnub;
    this.roomId = roomId;
    this.stateChannel = `game.${roomId}.state`;
    this.players = players;
    this.currentTurnIndex = 0;
    this.turnNumber = 0;
    this.turnTimeout = 30000; // 30 seconds per turn
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

    // Start turn timer
    this.turnTimer = setTimeout(() => {
      this.handleTurnTimeout();
    }, this.turnTimeout);
  }

  async submitAction(playerId, action) {
    if (playerId !== this.currentPlayer) {
      throw new Error('Not your turn');
    }

    clearTimeout(this.turnTimer);

    // Broadcast the action
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

    // Advance to next player
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

    // Skip to next player
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

### Turn-Based Game Example (Card Game)

```javascript
// Player submits a card play
async function playCard(pubnub, turnManager, cardId) {
  const action = {
    type: 'play-card',
    cardId,
    fromHand: true
  };

  await turnManager.submitAction(pubnub.getUUID(), action);
}

// All players listen for turn actions
pubnub.addListener({
  message: (event) => {
    const msg = event.message;

    switch (msg.type) {
      case 'turn-start':
        updateUI({
          activePlayer: msg.activePlayer,
          timeRemaining: msg.deadline - Date.now(),
          isMyTurn: msg.activePlayer === pubnub.getUUID()
        });
        break;

      case 'turn-action':
        animateAction(msg.action, msg.playerId);
        updateGameState(msg.action);
        break;

      case 'turn-timeout':
        showTimeoutMessage(msg.timedOutPlayer);
        break;
    }
  }
});
```

## Real-Time Action Game Patterns

### Client-Side Prediction with Reconciliation

```javascript
class PredictiveGameClient {
  constructor(pubnub, stateChannel) {
    this.pubnub = pubnub;
    this.stateChannel = stateChannel;
    this.localState = {};
    this.serverState = {};
    this.pendingInputs = [];
    this.inputSequence = 0;
  }

  // Apply input locally immediately (prediction)
  applyInput(input) {
    this.inputSequence++;
    input.sequence = this.inputSequence;
    input.timestamp = Date.now();

    // Apply locally for instant feedback
    this.applyToState(this.localState, input);

    // Send to server/host
    this.pubnub.publish({
      channel: this.stateChannel,
      message: {
        type: 'player-input',
        playerId: this.pubnub.getUUID(),
        input
      }
    });

    // Keep for reconciliation
    this.pendingInputs.push(input);
  }

  // When server confirms state, reconcile
  onServerUpdate(serverState, lastProcessedSequence) {
    this.serverState = serverState;

    // Remove acknowledged inputs
    this.pendingInputs = this.pendingInputs.filter(
      (input) => input.sequence > lastProcessedSequence
    );

    // Re-apply unacknowledged inputs on top of server state
    this.localState = JSON.parse(JSON.stringify(serverState));
    for (const input of this.pendingInputs) {
      this.applyToState(this.localState, input);
    }
  }

  applyToState(state, input) {
    const player = state.players[this.pubnub.getUUID()];
    if (!player) return;

    switch (input.type) {
      case 'move':
        player.position.x += input.dx;
        player.position.y += input.dy;
        break;
      case 'jump':
        player.velocity.y = -input.force;
        break;
    }
  }
}
```

### Entity Interpolation for Remote Players

```javascript
class EntityInterpolator {
  constructor() {
    this.entityBuffers = new Map(); // entityId -> position buffer
    this.interpolationDelay = 100;  // ms behind latest update
  }

  receiveUpdate(entityId, position, timestamp) {
    if (!this.entityBuffers.has(entityId)) {
      this.entityBuffers.set(entityId, []);
    }

    this.entityBuffers.get(entityId).push({
      position,
      timestamp
    });

    // Keep only the last 10 updates
    const buffer = this.entityBuffers.get(entityId);
    if (buffer.length > 10) {
      buffer.shift();
    }
  }

  getInterpolatedPosition(entityId) {
    const buffer = this.entityBuffers.get(entityId);
    if (!buffer || buffer.length < 2) {
      return buffer?.[0]?.position || null;
    }

    const renderTime = Date.now() - this.interpolationDelay;

    // Find two states to interpolate between
    for (let i = buffer.length - 1; i >= 1; i--) {
      if (buffer[i - 1].timestamp <= renderTime && buffer[i].timestamp >= renderTime) {
        const t = (renderTime - buffer[i - 1].timestamp) /
                  (buffer[i].timestamp - buffer[i - 1].timestamp);

        return {
          x: lerp(buffer[i - 1].position.x, buffer[i].position.x, t),
          y: lerp(buffer[i - 1].position.y, buffer[i].position.y, t)
        };
      }
    }

    // Fallback to latest known position
    return buffer[buffer.length - 1].position;
  }
}

function lerp(a, b, t) {
  return a + (b - a) * Math.max(0, Math.min(1, t));
}
```

## Anti-Cheat Validation via PubNub Functions

### Before Publish Trigger for Move Validation

```javascript
// PubNub Function: Before Publish on game.*.state channels
export default (request) => {
  const message = request.message;

  if (message.type === 'player-input' || message.type === 'state-delta') {
    const validation = validateGameAction(message);

    if (!validation.valid) {
      // Option 1: Block the message entirely
      // return request.abort('Cheat detected');

      // Option 2: Replace with a rejection message
      request.message = {
        type: 'action-rejected',
        originalType: message.type,
        playerId: message.playerId,
        reason: validation.reason,
        timestamp: Date.now()
      };
    }
  }

  return request.ok();
};

function validateGameAction(message) {
  // Validate movement speed
  if (message.input?.type === 'move') {
    const dx = Math.abs(message.input.dx || 0);
    const dy = Math.abs(message.input.dy || 0);
    const distance = Math.sqrt(dx * dx + dy * dy);
    const MAX_SPEED = 10; // units per tick

    if (distance > MAX_SPEED) {
      return { valid: false, reason: 'Movement speed exceeded' };
    }
  }

  // Validate damage values
  if (message.delta) {
    for (const [path, value] of Object.entries(message.delta)) {
      if (path.includes('.health') && typeof value === 'object') {
        if (value.operation === 'decrement' && value.value > 100) {
          return { valid: false, reason: 'Damage value out of range' };
        }
      }
    }
  }

  // Validate message frequency (rate limiting)
  if (message.timestamp) {
    const now = Date.now();
    if (message.timestamp > now + 5000) {
      return { valid: false, reason: 'Timestamp in the future' };
    }
  }

  return { valid: true };
}
```

### Server-Side Score Validation

```javascript
// PubNub Function: On Request handler for score submission
// Endpoint: POST /submit-score
export default (request, response) => {
  const db = require('kvstore');
  const body = JSON.parse(request.body);
  const { playerId, roomId, score, gameEvents } = body;

  // Replay game events to verify score
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

  // Store verified score
  return db.get(`leaderboard`).then((leaderboard) => {
    leaderboard = leaderboard || [];
    leaderboard.push({
      playerId,
      score: calculatedScore,
      roomId,
      timestamp: Date.now()
    });

    // Sort and keep top 100
    leaderboard.sort((a, b) => b.score - a.score);
    leaderboard = leaderboard.slice(0, 100);

    return db.set('leaderboard', leaderboard).then(() => {
      response.status = 200;
      return response.send({ verified: true, score: calculatedScore });
    });
  });
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
    // Use PubNub Function On Request to fetch from KV store
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

```javascript
class InGameChat {
  constructor(pubnub, roomId) {
    this.pubnub = pubnub;
    this.chatChannel = `game.${roomId}.chat`;
    this.messages = [];
  }

  connect(onMessage) {
    this.pubnub.subscribe({ channels: [this.chatChannel] });

    this.pubnub.addListener({
      message: (event) => {
        if (event.channel === this.chatChannel) {
          const msg = {
            senderId: event.message.senderId,
            text: event.message.text,
            timestamp: event.timetoken,
            team: event.message.team || null
          };
          this.messages.push(msg);
          onMessage(msg);
        }
      }
    });
  }

  async sendMessage(text, team = null) {
    await this.pubnub.publish({
      channel: this.chatChannel,
      message: {
        type: 'chat',
        senderId: this.pubnub.getUUID(),
        text,
        team
      }
    });
  }

  async sendQuickChat(phraseKey) {
    // Predefined phrases that work across languages
    await this.pubnub.publish({
      channel: this.chatChannel,
      message: {
        type: 'quick-chat',
        senderId: this.pubnub.getUUID(),
        phraseKey // e.g., 'nice_shot', 'good_game', 'need_help'
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

```javascript
class SpectatorManager {
  constructor(pubnub, roomId) {
    this.pubnub = pubnub;
    this.roomId = roomId;
    this.spectatorChannel = `game.${roomId}.spectator`;
    this.stateChannel = `game.${roomId}.state`;
  }

  // Spectators subscribe to a delayed, read-only feed
  async joinAsSpectator() {
    this.pubnub.subscribe({
      channels: [this.spectatorChannel]
    });

    // Request current game state snapshot
    await this.pubnub.publish({
      channel: this.stateChannel,
      message: {
        type: 'spectator-join',
        spectatorId: this.pubnub.getUUID()
      }
    });
  }

  // Host broadcasts to spectator channel with optional delay
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

## Game Architecture Comparison

| Architecture | Players | Latency Tolerance | PubNub Channels Needed | Example Games |
|-------------|---------|-------------------|----------------------|---------------|
| **Turn-Based** | 2-8 | High (seconds) | 2-3 per room | Chess, card games |
| **Real-Time Casual** | 2-16 | Medium (100-300ms) | 3-4 per room | Party games, trivia |
| **Real-Time Action** | 2-32 | Low (<100ms) | 4-5 per room | Shooters, racing |
| **Massive Lobby** | 50-1000 | Variable | 5-10+ (channel groups) | Battle royale lobby |
| **Persistent World** | 100+ | Medium | Zone-based channels | MMO, sandbox |

## Mobile Platform Patterns

### Swift (iOS) Game Room Join

```swift
import PubNub

class GameClient {
    let pubnub: PubNub

    init(userId: String) {
        let config = PubNubConfiguration(
            publishKey: "pub-c-...",
            subscribeKey: "sub-c-...",
            userId: userId
        )
        config.heartbeatInterval = 10
        config.presenceTimeout = 20
        self.pubnub = PubNub(configuration: config)
    }

    func joinRoom(roomId: String) {
        let channels = [
            "game.\(roomId)",
            "game.\(roomId).state",
            "game.\(roomId).chat"
        ]
        pubnub.subscribe(to: channels, withPresence: true)
    }
}
```

### Kotlin (Android) Game Room Join

```kotlin
import com.pubnub.api.PubNub
import com.pubnub.api.PNConfiguration
import com.pubnub.api.UserId

class GameClient(userId: String) {
    private val pubnub: PubNub

    init {
        val config = PNConfiguration(UserId(userId)).apply {
            publishKey = "pub-c-..."
            subscribeKey = "sub-c-..."
            heartbeatInterval = 10
            presenceTimeout = 20
        }
        pubnub = PubNub.create(config)
    }

    fun joinRoom(roomId: String) {
        pubnub.subscribe(
            channels = listOf(
                "game.$roomId",
                "game.$roomId.state",
                "game.$roomId.chat"
            ),
            withPresence = true
        )
    }
}
```

## Best Practices

1. **Keep matchmaking server-side** -- use PubNub Functions (On Request or After Publish) for matchmaking logic to prevent clients from manipulating queue position or pairing.

2. **Add turn timers for turn-based games** -- always enforce a maximum turn duration; skip or forfeit when a player exceeds the limit to keep the game progressing.

3. **Use a spectator delay** -- broadcast game state to spectators with a 3-5 second delay to prevent stream sniping in competitive games.

4. **Validate game actions on the server** -- use PubNub Functions Before Publish triggers to reject impossible actions (teleportation, excessive damage, invalid moves) before they reach other players.

5. **Implement quick-chat with phrase keys** -- instead of sending text strings, send localized phrase keys (e.g., `nice_shot`) so the message displays in each player's language.

6. **Store leaderboards in PubNub KV Store** -- use the built-in KV store in PubNub Functions for leaderboard persistence; broadcast updates to a leaderboard channel for real-time display.

7. **Design for graceful degradation** -- if a player disconnects in a turn-based game, auto-skip their turn; in action games, have AI take over temporarily.

8. **Separate chat from game state channels** -- game state messages should never be delayed by chat traffic; use dedicated channels for each concern.

9. **Use presence for player counts in lobbies** -- call `hereNow` to show live player counts in room listings rather than maintaining your own counters.

10. **Test matchmaking with varied player pools** -- simulate uneven rating distributions, players joining and leaving queues, and edge cases like exactly one player remaining in the queue.

11. **Rate-limit client publishes** -- enforce a maximum message rate on the client side to prevent accidental flooding; PubNub Functions can add server-side rate limiting as well.

12. **Keep mobile battery usage in mind** -- use longer heartbeat intervals on mobile when the game is in the background and reduce update frequency for non-critical data.
