# PubNub Game State Synchronization

**Canonical owner (S5):** Delta updates, sequence/version ordering, snapshots/resync, and stale-state protection — **reusable beyond gaming** (sport scoreboards, IoT telemetry, live dashboards). Vertical skills link here; keep domain state models locally.

## Overview

PubNub delivers ordered messages per channel. Use that to publish `state-delta` payloads with sequence numbers, then snapshot/resync when a client reconnects or detects a gap.

## State Synchronization Models

### Authoritative Server with PubNub Functions

Use the [server-authoritative Before-Publish pattern](../../pubnub-functions/references/functions-patterns.md) to validate `player-action` messages before delivery. **Game-specific delta:** validate move distance, cooldowns, and game rules in `validateAction`; transform accepted actions into `state-update` payloads with `serverTimestamp`.

### Host-Authoritative Model

One player hosts: others publish `player-action` on `game.{roomId}.state`; the host publishes `authoritative-update` with `serverSeq` and a delta.

```javascript
class HostAuthoritativeSync {
  constructor(pubnub, roomId, isHost) {
    this.pubnub = pubnub;
    this.roomId = roomId;
    this.isHost = isHost;
    this.stateChannel = `game.${roomId}.state`;
    this.gameState = {};
    this.sequenceNumber = 0;
  }

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

  processActionAsHost(actionMsg) {
    if (!this.isHost) return;

    const { action, playerId, clientSeq } = actionMsg;
    const result = this.validateAndApply(action, playerId);

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
    // Domain rules live in the game; return a path-keyed delta for PubNub.
    return { delta: {} };
  }
}
```

## Delta Updates

Sending only changed state properties instead of the full game state reduces message size and bandwidth consumption. This is critical for staying under PubNub's message size limit — retrieve the current cap via **`how_to`** (`calculate-message-payload-size`).

### Delta Update Structure

```javascript
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

Apply path-keyed `delta` objects in `seq` order. Drop or snapshot-resync if a sequence gap appears.

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
```

## Conflict Resolution

Prefer **server- or host-authoritative** updates on `game.{roomId}.state` for competitive fields (health, score). Per-channel PubNub ordering plus `seq` is the durable stale-state protection: ignore a delta whose `seq` is not strictly greater than the last applied sequence; request a snapshot on gaps. Do not invent a second ordering scheme across channels — PubNub does not guarantee cross-channel order.

## State Snapshot and Recovery

Players who disconnect and rejoin need the full current game state. State snapshots solve this by allowing any client (or the host) to send a complete state to the reconnecting player.

### Snapshot Request/Response

```javascript
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

function applySnapshot(msg, localState) {
  if (msg.type === 'snapshot-response' && msg.targetPlayer === localPlayerId) {
    Object.assign(localState, msg.fullState);
    localSequenceNumber = msg.atSequence;
  }
}
```

### Recovery after disconnect

Follow the canonical [offline catch-up flow](../../pubnub-history/references/offline-catch-up.md) with [dedup-on-merge](../../pubnub-reliability/references/dedup-on-merge.md). **Game-specific delta:** when replaying history, apply only `state-delta` messages to `gameState` in timetoken order; request a full snapshot if sequence gaps remain.

## Handling Player Disconnections Mid-Game

On presence `timeout` for `game.{roomId}`, publish `game-paused` / `player-disconnected` on the room channel and stop publishing ticks until reconnect. On `PNReconnectedCategory` or presence `join`, request a snapshot on `.state` (see Snapshot Request/Response). Connection-manager publish shapes live in [gaming-setup.md](gaming-setup.md).

## Best Practices

1. **Send delta updates, not full state** -- only publish properties that actually changed to reduce message size and bandwidth.

2. **Use sequence numbers on every state message** -- sequence numbers allow receivers to detect missed messages and request resynchronization.

3. **Batch frequent updates** -- accumulate changes over a short window (30-50ms) and send them as a single publish call.

4. **Implement state snapshots** -- provide a mechanism for the host or server to send the full game state to reconnecting players or new spectators.

5. **Use Message Persistence for recovery** — follow [offline catch-up](../../pubnub-history/references/offline-catch-up.md); apply game deltas after merge with dedup.

6. **Keep state messages within PubNub's message size limit** — retrieve the current cap via **`how_to`**. If your full state exceeds it, use delta updates or split into multiple messages.

7. **Handle out-of-order processing gracefully** -- even though PubNub guarantees per-channel ordering, cross-channel messages may arrive in any order. Use timestamps or sequence numbers for cross-channel coordination.
