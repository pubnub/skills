# PubNub Game State Synchronization

## Overview

Game state synchronization is the core challenge of multiplayer game networking. PubNub's publish/subscribe model provides reliable, ordered message delivery per channel, making it well suited for both authoritative and peer-to-peer state models. This guide covers delta updates, conflict resolution, latency compensation, and state recovery patterns.

## State Synchronization Models

### Comparison of Approaches

| Model | Description | Best For | Latency | Cheat Resistance |
|-------|-------------|----------|---------|-----------------|
| **Authoritative Server** | Single source of truth validates all actions | Competitive games, ranked play | Higher (round-trip) | High |
| **Peer-to-Peer** | Players share state directly | Casual games, co-op | Lower (one-way) | Low |
| **Host-Authoritative** | One player acts as the server | Small lobbies, party games | Medium | Medium |
| **Lockstep** | All players process same inputs in order | Turn-based, RTS | Variable | Medium |

### Authoritative Server with PubNub Functions

```javascript
// PubNub Function (Before Publish handler) acts as authoritative server
// This runs on PubNub's edge network before the message is delivered
export default (request) => {
  const message = request.message;

  if (message.type === 'player-action') {
    // Validate the action server-side
    const validation = validateAction(message.action, message.playerId);

    if (!validation.valid) {
      // Reject the message - it will not be published
      request.message = {
        type: 'action-rejected',
        playerId: message.playerId,
        reason: validation.reason
      };
    } else {
      // Apply the action and compute new state
      request.message = {
        type: 'state-update',
        action: message.action,
        result: validation.result,
        serverTimestamp: Date.now()
      };
    }
  }

  return request.ok();
};

function validateAction(action, playerId) {
  // Server-side validation logic
  if (action.type === 'move' && action.distance > MAX_MOVE_DISTANCE) {
    return { valid: false, reason: 'Invalid move distance' };
  }
  return { valid: true, result: computeResult(action) };
}
```

### Host-Authoritative Model

```javascript
class HostAuthoritativeSync {
  constructor(pubnub, roomId, isHost) {
    this.pubnub = pubnub;
    this.roomId = roomId;
    this.isHost = isHost;
    this.stateChannel = `game.${roomId}.state`;
    this.gameState = {};
    this.pendingActions = [];
    this.sequenceNumber = 0;
  }

  // Players send actions to the host
  async sendAction(action) {
    await this.pubnub.publish({
      channel: this.stateChannel,
      message: {
        type: 'player-action',
        playerId: this.pubnub.getUserId(),
        action,
        clientSeq: ++this.sequenceNumber,
        timestamp: Date.now()
      }
    });
  }

  // Host processes actions and broadcasts authoritative state
  processActionAsHost(actionMsg) {
    if (!this.isHost) return;

    const { action, playerId, clientSeq } = actionMsg;
    const result = this.validateAndApply(action, playerId);

    // Broadcast the authoritative result to all players
    this.pubnub.publish({
      channel: this.stateChannel,
      message: {
        type: 'authoritative-update',
        serverSeq: ++this.sequenceNumber,
        delta: result.delta,
        processedAction: { playerId, clientSeq },
        timestamp: Date.now()
      }
    });
  }

  validateAndApply(action, playerId) {
    // Validate and compute state change
    const delta = {};

    switch (action.type) {
      case 'move':
        const player = this.gameState.players[playerId];
        if (isValidMove(player, action.target)) {
          delta[`players.${playerId}.position`] = action.target;
        }
        break;

      case 'attack':
        const damage = calculateDamage(action);
        delta[`players.${action.targetId}.health`] = Math.max(
          0,
          this.gameState.players[action.targetId].health - damage
        );
        break;
    }

    // Apply delta to host state
    applyDelta(this.gameState, delta);
    return { delta };
  }
}
```

## Delta Updates

Sending only changed state properties instead of the full game state reduces message size and bandwidth consumption. This is critical for keeping messages under PubNub's 32 KB limit.

### Delta Update Structure

```javascript
// WRONG: Sending full state every update
await pubnub.publish({
  channel: stateChannel,
  message: {
    type: 'full-state',
    state: entireGameState  // Could be very large
  }
});

// CORRECT: Send only what changed
await pubnub.publish({
  channel: stateChannel,
  message: {
    type: 'state-delta',
    seq: 142,
    senderId: 'player-abc',
    delta: {
      'players.player-abc.position': { x: 150, y: 320 },
      'players.player-abc.facing': 'north'
    },
    timestamp: Date.now()
  }
});
```

### Applying Deltas

```javascript
function applyDelta(state, delta) {
  for (const [path, value] of Object.entries(delta)) {
    setNestedValue(state, path, value);
  }
}

function setNestedValue(obj, path, value) {
  const keys = path.split('.');
  let current = obj;

  for (let i = 0; i < keys.length - 1; i++) {
    if (!(keys[i] in current)) {
      current[keys[i]] = {};
    }
    current = current[keys[i]];
  }

  current[keys[keys.length - 1]] = value;
}

function getNestedValue(obj, path) {
  const keys = path.split('.');
  let current = obj;

  for (const key of keys) {
    if (current === undefined || current === null) return undefined;
    current = current[key];
  }

  return current;
}
```

### Batching Delta Updates

```javascript
class DeltaBatcher {
  constructor(pubnub, stateChannel, options = {}) {
    this.pubnub = pubnub;
    this.stateChannel = stateChannel;
    this.batchInterval = options.batchInterval || 50; // ms
    this.pendingDeltas = {};
    this.sequenceNumber = 0;
    this.timer = null;
  }

  queueDelta(path, value) {
    this.pendingDeltas[path] = value;

    if (!this.timer) {
      this.timer = setTimeout(() => this.flush(), this.batchInterval);
    }
  }

  async flush() {
    if (Object.keys(this.pendingDeltas).length === 0) return;

    const delta = { ...this.pendingDeltas };
    this.pendingDeltas = {};
    this.timer = null;

    await this.pubnub.publish({
      channel: this.stateChannel,
      message: {
        type: 'state-delta',
        seq: ++this.sequenceNumber,
        senderId: this.pubnub.getUserId(),
        delta,
        timestamp: Date.now()
      }
    });
  }

  destroy() {
    if (this.timer) {
      clearTimeout(this.timer);
      this.timer = null;
    }
  }
}

// Usage
const batcher = new DeltaBatcher(pubnub, stateChannel, { batchInterval: 50 });

// These will be batched into a single publish
batcher.queueDelta('players.abc.position.x', 100);
batcher.queueDelta('players.abc.position.y', 200);
batcher.queueDelta('players.abc.health', 85);
```

## Conflict Resolution

When multiple players update the same state simultaneously, conflicts can arise. Different strategies are appropriate for different types of game data.

### Conflict Resolution Strategies

| Strategy | How It Works | Best For |
|----------|-------------|----------|
| **Last-Write-Wins** | Latest timestamp overwrites | Player positions, non-critical state |
| **Server-Authoritative** | Server decides final state | Competitive scoring, health |
| **Operational Transform** | Merge concurrent operations | Collaborative building games |
| **Priority-Based** | Higher priority player wins | Turn-based actions |
| **Accumulative** | All changes stack | Damage, score increments |

### Last-Write-Wins Implementation

```javascript
class LWWConflictResolver {
  constructor() {
    this.stateTimestamps = {};
  }

  applyUpdate(state, delta, timestamp) {
    const applied = {};

    for (const [path, value] of Object.entries(delta)) {
      const existingTimestamp = this.stateTimestamps[path] || 0;

      if (timestamp >= existingTimestamp) {
        setNestedValue(state, path, value);
        this.stateTimestamps[path] = timestamp;
        applied[path] = value;
      }
      // Otherwise, this update is stale and ignored
    }

    return applied;
  }
}
```

### Accumulative Resolution (for Scores/Damage)

```javascript
class AccumulativeResolver {
  applyUpdate(state, delta) {
    for (const [path, change] of Object.entries(delta)) {
      if (change.operation === 'increment') {
        const current = getNestedValue(state, path) || 0;
        setNestedValue(state, path, current + change.value);
      } else if (change.operation === 'decrement') {
        const current = getNestedValue(state, path) || 0;
        setNestedValue(state, path, Math.max(0, current - change.value));
      }
    }
  }
}

// Usage: sending accumulative updates
await pubnub.publish({
  channel: stateChannel,
  message: {
    type: 'state-delta',
    delta: {
      'players.player-xyz.score': { operation: 'increment', value: 10 },
      'players.player-xyz.health': { operation: 'decrement', value: 25 }
    },
    timestamp: Date.now()
  }
});
```

## Frame/Tick Synchronization

For real-time action games, synchronizing updates on a fixed tick rate ensures consistent game simulation across all clients.

### Fixed Tick Rate Publisher

```javascript
class GameTickManager {
  constructor(pubnub, stateChannel, tickRate = 20) {
    this.pubnub = pubnub;
    this.stateChannel = stateChannel;
    this.tickRate = tickRate;  // Ticks per second
    this.tickInterval = 1000 / tickRate;
    this.currentTick = 0;
    this.localInputBuffer = [];
    this.intervalId = null;
  }

  start() {
    this.intervalId = setInterval(() => {
      this.processTick();
    }, this.tickInterval);
  }

  stop() {
    if (this.intervalId) {
      clearInterval(this.intervalId);
      this.intervalId = null;
    }
  }

  addInput(input) {
    this.localInputBuffer.push({
      ...input,
      tick: this.currentTick
    });
  }

  async processTick() {
    this.currentTick++;

    if (this.localInputBuffer.length === 0) return;

    const inputs = [...this.localInputBuffer];
    this.localInputBuffer = [];

    await this.pubnub.publish({
      channel: this.stateChannel,
      message: {
        type: 'tick-input',
        tick: this.currentTick,
        playerId: this.pubnub.getUserId(),
        inputs,
        timestamp: Date.now()
      }
    });
  }
}
```

### Input Processing on Receiver

```javascript
class TickReceiver {
  constructor(gameState) {
    this.gameState = gameState;
    this.inputQueue = new Map(); // tick -> [inputs]
    this.lastProcessedTick = 0;
  }

  receiveInput(tickMsg) {
    const { tick, playerId, inputs } = tickMsg;

    if (!this.inputQueue.has(tick)) {
      this.inputQueue.set(tick, []);
    }

    this.inputQueue.get(tick).push({ playerId, inputs });
    this.processReadyTicks();
  }

  processReadyTicks() {
    while (this.inputQueue.has(this.lastProcessedTick + 1)) {
      this.lastProcessedTick++;
      const tickInputs = this.inputQueue.get(this.lastProcessedTick);
      this.inputQueue.delete(this.lastProcessedTick);

      for (const { playerId, inputs } of tickInputs) {
        for (const input of inputs) {
          this.applyInput(playerId, input);
        }
      }
    }
  }

  applyInput(playerId, input) {
    // Game-specific input application
    switch (input.type) {
      case 'move':
        this.gameState.players[playerId].position = input.position;
        break;
      case 'shoot':
        this.gameState.projectiles.push({
          owner: playerId,
          position: input.origin,
          direction: input.direction,
          tick: input.tick
        });
        break;
    }
  }
}
```

## State Snapshot and Recovery

Players who disconnect and rejoin need the full current game state. State snapshots solve this by allowing any client (or the host) to send a complete state to the reconnecting player.

### Snapshot Request/Response

```javascript
// Reconnecting player requests current state
async function requestStateSnapshot(pubnub, stateChannel) {
  await pubnub.publish({
    channel: stateChannel,
    message: {
      type: 'snapshot-request',
      requesterId: pubnub.getUserId(),
      timestamp: Date.now()
    }
  });
}

// Host (or designated authority) responds with full state
function handleSnapshotRequest(pubnub, stateChannel, gameState, requesterId) {
  pubnub.publish({
    channel: stateChannel,
    message: {
      type: 'snapshot-response',
      targetPlayer: requesterId,
      fullState: gameState,
      atSequence: currentSequenceNumber,
      timestamp: Date.now()
    }
  });
}

// Reconnecting player receives and applies snapshot
function applySnapshot(msg, localState) {
  if (msg.type === 'snapshot-response' && msg.targetPlayer === localPlayerId) {
    Object.assign(localState, msg.fullState);
    localSequenceNumber = msg.atSequence;
    console.log(`State recovered at sequence ${msg.atSequence}`);
  }
}
```

### Using Message Persistence for Recovery

```javascript
// Fetch missed messages from PubNub history
async function recoverMissedUpdates(pubnub, stateChannel, lastKnownTimetoken) {
  const result = await pubnub.fetchMessages({
    channels: [stateChannel],
    start: lastKnownTimetoken,
    count: 100
  });

  const messages = result.channels[stateChannel] || [];

  // Apply missed deltas in order
  for (const msg of messages) {
    if (msg.message.type === 'state-delta') {
      applyDelta(gameState, msg.message.delta);
    }
  }

  return messages.length;
}
```

## Handling Player Disconnections Mid-Game

### Pause-on-Disconnect Strategy

```javascript
class DisconnectHandler {
  constructor(pubnub, roomId, gameTickManager) {
    this.pubnub = pubnub;
    this.roomId = roomId;
    this.tickManager = gameTickManager;
    this.disconnectedPlayers = new Set();
    this.pauseTimeout = 5000; // 5s grace before pausing
    this.pauseTimers = new Map();
  }

  onPlayerTimeout(playerId) {
    this.disconnectedPlayers.add(playerId);

    // Start a pause timer
    const timer = setTimeout(() => {
      if (this.disconnectedPlayers.has(playerId)) {
        this.pauseGame(playerId);
      }
    }, this.pauseTimeout);

    this.pauseTimers.set(playerId, timer);
  }

  onPlayerReconnect(playerId) {
    this.disconnectedPlayers.delete(playerId);

    const timer = this.pauseTimers.get(playerId);
    if (timer) {
      clearTimeout(timer);
      this.pauseTimers.delete(playerId);
    }

    // Resume if game was paused for this player
    if (this.isPaused) {
      this.resumeGame();
    }
  }

  pauseGame(disconnectedPlayerId) {
    this.isPaused = true;
    this.tickManager.stop();

    this.pubnub.publish({
      channel: `game.${this.roomId}`,
      message: {
        type: 'game-paused',
        reason: 'player-disconnect',
        disconnectedPlayer: disconnectedPlayerId,
        timestamp: Date.now()
      }
    });
  }

  resumeGame() {
    this.isPaused = false;
    this.tickManager.start();

    this.pubnub.publish({
      channel: `game.${this.roomId}`,
      message: {
        type: 'game-resumed',
        timestamp: Date.now()
      }
    });
  }
}
```

### AI Takeover Strategy

```javascript
// Replace disconnected player with AI until they return
function activateAI(gameState, disconnectedPlayerId) {
  gameState.players[disconnectedPlayerId].controlledBy = 'ai';

  // Simple AI tick processing
  function aiTick(playerState) {
    // Basic AI behavior
    return {
      type: 'move',
      position: calculateSafePosition(playerState, gameState)
    };
  }

  return aiTick;
}

function deactivateAI(gameState, reconnectedPlayerId) {
  gameState.players[reconnectedPlayerId].controlledBy = 'human';
}
```

## State Compression

For games with large state, compress updates to stay within PubNub's message size limit.

```javascript
// Simple compression: remove default/unchanged values
function compressState(fullState, defaultState) {
  const compressed = {};

  for (const [key, value] of Object.entries(fullState)) {
    if (JSON.stringify(value) !== JSON.stringify(defaultState[key])) {
      compressed[key] = value;
    }
  }

  return compressed;
}

// Binary encoding for position data (reduce message size)
function encodePositions(players) {
  // Pack positions into a compact format
  const encoded = [];
  for (const player of Object.values(players)) {
    encoded.push(
      Math.round(player.position.x),
      Math.round(player.position.y),
      Math.round(player.rotation * 100) / 100
    );
  }
  return encoded;
}

function decodePositions(encoded, playerIds) {
  const positions = {};
  for (let i = 0; i < playerIds.length; i++) {
    positions[playerIds[i]] = {
      x: encoded[i * 3],
      y: encoded[i * 3 + 1],
      rotation: encoded[i * 3 + 2]
    };
  }
  return positions;
}
```

## Best Practices

1. **Send delta updates, not full state** -- only publish properties that actually changed to reduce message size and bandwidth.

2. **Use sequence numbers on every state message** -- sequence numbers allow receivers to detect missed messages and request resynchronization.

3. **Batch frequent updates** -- for fast-paced games, accumulate changes over a short window (30-50ms) and send them as a single publish call.

4. **Implement state snapshots** -- provide a mechanism for the host or server to send the full game state to reconnecting players or new spectators.

5. **Choose the right sync model for your game type** -- turn-based games work well with lockstep, casual games with peer-to-peer, competitive games with server-authoritative.

6. **Use Message Persistence for recovery** -- enable message storage so reconnecting clients can fetch missed updates from PubNub history.

7. **Keep state messages under 32 KB** -- PubNub's maximum message size is 32 KB. If your full state exceeds this, you must use delta updates or split into multiple messages.

8. **Handle out-of-order processing gracefully** -- even though PubNub guarantees per-channel ordering, cross-channel messages may arrive in any order. Use timestamps or sequence numbers for cross-channel coordination.

9. **Test with simulated packet loss** -- add artificial delays and message drops during development to verify your conflict resolution and recovery code handles real-world conditions.

10. **Separate authoritative state from cosmetic state** -- game-critical data (health, score, positions) should use conflict resolution, while cosmetic data (animations, particles) can use fire-and-forget.
