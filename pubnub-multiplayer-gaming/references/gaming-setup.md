<!-- xrefs-injected -->

> **Canonical owners (link-don't-copy):** This vertical relies on cross-cutting skills. Always link to the canonical owner instead of duplicating. Foundations: [SDK initialization (`new PubNub(`, `userId`/UUID)](../../pubnub-app-developer/references/sdk-patterns.md), [pub/sub basics (`pubnub.publish(`, `pubnub.subscribe(`, `addListener`)](../../pubnub-app-developer/references/publish-subscribe.md), [channel naming](../../pubnub-app-developer/references/channels.md), [message filters](../../pubnub-app-developer/references/message-filters.md), [SDK upgrades](../../pubnub-app-developer/references/sdk-upgrades.md), [REST API](../../pubnub-app-developer/references/rest-api.md). Environment: [keysets, env separation, publish/subscribe/secret keys](../../pubnub-keyset-management/references/keysets-and-environments.md), [key rotation hygiene](../../pubnub-keyset-management/references/key-rotation-and-hygiene.md), [demo keys](../../pubnub-keyset-management/references/demo-keys.md), [custom origin](../../pubnub-keyset-management/references/custom-origin.md). Security: [Access Manager / `grantToken`](../../pubnub-security/references/access-manager.md), [AES-256 / message encryption](../../pubnub-security/references/encryption.md), [IP allowlisting](../../pubnub-security/references/ip-whitelisting.md), [DoS mitigation](../../pubnub-security/references/dos-mitigation.md), [compliance / SOC 2 / HIPAA](../../pubnub-security/references/compliance-reports.md). Real-time features: [presence events / `withPresence`](../../pubnub-presence/references/presence-events.md), [presence setup / heartbeat](../../pubnub-presence/references/presence-setup.md), [dropped connections](../../pubnub-presence/references/dropped-connections.md), [multi-device sync](../../pubnub-presence/references/multi-device-sync.md). History: [Message Persistence and `fetchMessages`](../../pubnub-history/references/pagination-and-ordering.md), [offline catch-up](../../pubnub-history/references/offline-catch-up.md), [retention](../../pubnub-history/references/retention-and-storage.md). App Context: [users / user metadata](../../pubnub-app-context/references/users.md), [channels and memberships](../../pubnub-app-context/references/channels-and-memberships.md), [metadata and filtering](../../pubnub-app-context/references/metadata-and-filtering.md). Functions: [Before/After Publish, `request.ok()`/`request.abort()`](../../pubnub-functions/references/functions-basics.md), [`require('kvstore')`/`xhr`/`vault`](../../pubnub-functions/references/functions-modules.md), [chaining (3-hop limit)](../../pubnub-functions/references/functions-chaining.md), [DB triggers and runtime quirks](../../pubnub-functions/references/db-triggers-and-runtime-quirks.md), [common patterns](../../pubnub-functions/references/functions-patterns.md). Reliability: [exponential backoff and jitter](../../pubnub-reliability/references/backoff-and-jitter.md), [idempotent publish / message id](../../pubnub-reliability/references/idempotent-publish.md), [dedup on merge](../../pubnub-reliability/references/dedup-on-merge.md), [queue and retry](../../pubnub-reliability/references/queue-and-retry.md), [schema version](../../pubnub-reliability/references/schema-versioning.md). Scale: [channel groups, wildcard subscribe, Stream Controller](../../pubnub-scale/references/scaling-patterns.md), [performance tuning](../../pubnub-scale/references/performance.md), [10K+ live events](../../pubnub-scale/references/large-events.md). Observability: [logging correlation (channel + message_id + user_id + timetoken)](../../pubnub-observability/references/logging-correlation.md), [test pyramid](../../pubnub-observability/references/test-pyramid.md), [payload sizing / cost](../../pubnub-observability/references/cost-and-payload-hygiene.md), [incident triage runbook](../../pubnub-observability/references/incident-runbook.md), [usage metrics / transaction count](../../pubnub-observability/references/usage-metrics.md). Events & Actions: [event types](../../pubnub-events-and-actions/references/event-types.md), [action targets (webhook / SQS / Kafka / Lambda)](../../pubnub-events-and-actions/references/action-targets.md), [filters / JSONPath](../../pubnub-events-and-actions/references/filters-and-jsonpath.md). Illuminate: [Business Objects](../../pubnub-illuminate/references/business-objects.md), [Metrics](../../pubnub-illuminate/references/metrics.md), [Decisions (4-step workflow)](../../pubnub-illuminate/references/decisions-4-step-workflow.md), [Queries](../../pubnub-illuminate/references/queries-adhoc-vs-saved.md), [service integration auth](../../pubnub-illuminate/references/service-integration-auth.md). Chat: [Chat SDK setup](../../pubnub-chat/references/chat-setup.md), [message actions / reactions](../../pubnub-chat/references/message-actions.md), [file sharing / `sendFile`](../../pubnub-chat/references/file-sharing.md), [threading](../../pubnub-chat/references/threading.md). Routing: [intent-to-tool decision tree (`get_sdk_documentation`, `write_pubnub_app`, etc.)](../../pubnub-choose-docs-path/references/intent-to-tool.md).


# PubNub Multiplayer Gaming Setup

## Installation

### JavaScript/TypeScript

```bash
npm install pubnub
# or
yarn add pubnub
```

### Swift (iOS)

```swift
// Package.swift or CocoaPods
// SPM: https://github.com/pubnub/swift
// CocoaPods: pod 'PubNubSwift', '~> 7.0'
```

### Kotlin (Android)

```kotlin
// build.gradle
implementation("com.pubnub:pubnub-kotlin:9.0.0")
```

### Prerequisites

- PubNub account with Publish and Subscribe keys from the Admin Portal
- **Presence** feature enabled on your keyset
- **App Context** (Objects) enabled for storing room and player metadata
- **Message Persistence** enabled if you need game state recovery
- Optionally: **PubNub Functions** enabled for server-side validation

## Basic Initialization

```javascript
import PubNub from 'pubnub';

const pubnub = new PubNub({
  publishKey: 'pub-c-...',
  subscribeKey: 'sub-c-...',
  userId: 'player-unique-id'
});
```

## Gaming-Optimized Configuration

```javascript
const pubnub = new PubNub({
  publishKey: 'pub-c-...',
  subscribeKey: 'sub-c-...',
  userId: playerId,

  // Presence: detect disconnects quickly
  presenceTimeout: 20,          // Seconds before player is marked offline
  heartbeatInterval: 10,        // Heartbeat frequency in seconds

  // Reconnection: recover from network drops
  restore: true,                // Automatically resubscribe on reconnect
  retryConfiguration: PubNub.LinearRetryPolicy({
    delay: 1,                   // 1-second delay between retries
    maximumRetry: 10            // Retry up to 10 times
  }),

  // Optional: Access Manager
  authKey: 'auth-token-from-server',

  // Optional: message encryption
  cipherKey: 'shared-game-secret'
});
```

## Channel Architecture for Games

PubNub channels serve as the backbone of multiplayer game networking. A well-designed channel structure keeps traffic organized and minimizes unnecessary message delivery.

### Channel Naming Conventions

| Channel Pattern | Purpose | Example |
|----------------|---------|---------|
| `game-lobby` | Global lobby for browsing rooms | `game-lobby` |
| `game-lobby.{type}` | Lobby filtered by game type | `game-lobby.chess` |
| `game.{roomId}` | Main game room channel | `game.xk7f9a` |
| `game.{roomId}.state` | Game state sync channel | `game.xk7f9a.state` |
| `game.{roomId}.chat` | In-game chat for that room | `game.xk7f9a.chat` |
| `game.{roomId}.spectator` | Spectator broadcast channel | `game.xk7f9a.spectator` |
| `player.{playerId}` | Private player notifications | `player.abc123` |
| `matchmaking.{queue}` | Matchmaking queue channel | `matchmaking.ranked` |

### Channel Hierarchy Diagram

```
game-lobby                     (browse available rooms)
  game-lobby.chess             (browse chess rooms)
  game-lobby.trivia            (browse trivia rooms)

game.{roomId}                  (room control: join, leave, start, end)
  game.{roomId}.state          (game state delta updates)
  game.{roomId}.chat           (in-game text chat)
  game.{roomId}.spectator      (spectator-only broadcast)

player.{playerId}              (personal invites, match found alerts)

matchmaking.ranked             (ranked matchmaking queue)
matchmaking.casual             (casual matchmaking queue)
```

## Game Room Lifecycle

### Room States

| State | Description | Transitions To |
|-------|-------------|---------------|
| `waiting` | Room created, waiting for players | `ready`, `cancelled` |
| `ready` | Minimum players joined, can start | `in-progress`, `cancelled` |
| `in-progress` | Game is actively running | `finished`, `paused` |
| `paused` | Game paused (player disconnect) | `in-progress`, `cancelled` |
| `finished` | Game completed normally | (terminal) |
| `cancelled` | Room closed before finish | (terminal) |

### Creating a Game Room

```javascript
async function createGameRoom(pubnub, hostPlayerId, config) {
  const roomId = generateRoomId();
  const roomChannel = `game.${roomId}`;
  const stateChannel = `game.${roomId}.state`;
  const chatChannel = `game.${roomId}.chat`;

  // Store room metadata
  await pubnub.objects.setChannelMetadata({
    channel: roomChannel,
    data: {
      name: config.roomName || `Room ${roomId}`,
      description: JSON.stringify({
        host: hostPlayerId,
        gameType: config.gameType,
        maxPlayers: config.maxPlayers || 4,
        minPlayers: config.minPlayers || 2,
        status: 'waiting',
        isPrivate: config.isPrivate || false,
        createdAt: Date.now()
      })
    }
  });

  // Host subscribes to all room channels
  pubnub.subscribe({
    channels: [roomChannel, stateChannel, chatChannel],
    withPresence: true
  });

  // Announce room in lobby (unless private)
  if (!config.isPrivate) {
    await pubnub.publish({
      channel: `game-lobby.${config.gameType}`,
      message: {
        type: 'room-created',
        roomId,
        host: hostPlayerId,
        gameType: config.gameType,
        maxPlayers: config.maxPlayers || 4,
        roomName: config.roomName
      }
    });
  }

  return { roomId, roomChannel, stateChannel, chatChannel };
}

function generateRoomId() {
  return `${Date.now().toString(36)}-${Math.random().toString(36).slice(2, 8)}`;
}
```

### Joining a Game Room

```javascript
async function joinGameRoom(pubnub, playerId, roomId) {
  const roomChannel = `game.${roomId}`;
  const stateChannel = `game.${roomId}.state`;
  const chatChannel = `game.${roomId}.chat`;

  // Fetch room metadata to check status
  const { data } = await pubnub.objects.getChannelMetadata({
    channel: roomChannel
  });

  const roomInfo = JSON.parse(data.description);

  if (roomInfo.status !== 'waiting' && roomInfo.status !== 'ready') {
    throw new Error(`Cannot join room in "${roomInfo.status}" state`);
  }

  // Check current player count via presence
  const presenceResult = await pubnub.hereNow({
    channels: [roomChannel],
    includeUUIDs: true
  });

  const currentCount = presenceResult.channels[roomChannel]?.occupancy || 0;

  if (currentCount >= roomInfo.maxPlayers) {
    throw new Error('Room is full');
  }

  // Subscribe to room channels
  pubnub.subscribe({
    channels: [roomChannel, stateChannel, chatChannel],
    withPresence: true
  });

  // Announce join
  await pubnub.publish({
    channel: roomChannel,
    message: {
      type: 'player-joined',
      playerId,
      timestamp: Date.now()
    }
  });

  // Update room status if enough players
  if (currentCount + 1 >= roomInfo.minPlayers) {
    roomInfo.status = 'ready';
    await pubnub.objects.setChannelMetadata({
      channel: roomChannel,
      data: {
        description: JSON.stringify(roomInfo)
      }
    });
  }

  return { roomChannel, stateChannel, chatChannel, roomInfo };
}
```

### Starting a Game

```javascript
async function startGame(pubnub, roomId, hostPlayerId) {
  const roomChannel = `game.${roomId}`;
  const stateChannel = `game.${roomId}.state`;

  // Verify caller is host
  const { data } = await pubnub.objects.getChannelMetadata({
    channel: roomChannel
  });
  const roomInfo = JSON.parse(data.description);

  if (roomInfo.host !== hostPlayerId) {
    throw new Error('Only the host can start the game');
  }

  if (roomInfo.status !== 'ready') {
    throw new Error('Not enough players to start');
  }

  // Get all players in the room
  const presenceResult = await pubnub.hereNow({
    channels: [roomChannel],
    includeUUIDs: true
  });

  const players = presenceResult.channels[roomChannel]?.occupants.map(o => o.uuid) || [];

  // Update room status
  roomInfo.status = 'in-progress';
  roomInfo.startedAt = Date.now();
  roomInfo.players = players;

  await pubnub.objects.setChannelMetadata({
    channel: roomChannel,
    data: { description: JSON.stringify(roomInfo) }
  });

  // Broadcast game start with initial state
  await pubnub.publish({
    channel: stateChannel,
    message: {
      type: 'game-start',
      players,
      initialState: buildInitialGameState(players, roomInfo.gameType),
      timestamp: Date.now()
    }
  });

  // Remove room from lobby
  await pubnub.publish({
    channel: `game-lobby.${roomInfo.gameType}`,
    message: {
      type: 'room-started',
      roomId
    }
  });

  return { players, roomInfo };
}

function buildInitialGameState(players, gameType) {
  return {
    players: players.map((id, index) => ({
      id,
      index,
      score: 0,
      isActive: true
    })),
    turn: 0,
    round: 1,
    gameType,
    startedAt: Date.now()
  };
}
```

### Ending a Game

```javascript
async function endGame(pubnub, roomId, results) {
  const roomChannel = `game.${roomId}`;
  const stateChannel = `game.${roomId}.state`;

  // Broadcast game results
  await pubnub.publish({
    channel: stateChannel,
    message: {
      type: 'game-end',
      results,
      timestamp: Date.now()
    }
  });

  // Update room metadata
  const { data } = await pubnub.objects.getChannelMetadata({
    channel: roomChannel
  });
  const roomInfo = JSON.parse(data.description);
  roomInfo.status = 'finished';
  roomInfo.endedAt = Date.now();
  roomInfo.results = results;

  await pubnub.objects.setChannelMetadata({
    channel: roomChannel,
    data: { description: JSON.stringify(roomInfo) }
  });

  // Remove room from lobby listing
  await pubnub.publish({
    channel: `game-lobby.${roomInfo.gameType}`,
    message: {
      type: 'room-closed',
      roomId
    }
  });
}
```

## Lobby Management

### Browsing Available Rooms

```javascript
function setupLobby(pubnub, gameType) {
  const lobbyChannel = gameType ? `game-lobby.${gameType}` : 'game-lobby';
  const availableRooms = new Map();

  pubnub.subscribe({ channels: [lobbyChannel] });

  pubnub.addListener({
    message: (event) => {
      if (event.channel !== lobbyChannel) return;

      const msg = event.message;

      switch (msg.type) {
        case 'room-created':
          availableRooms.set(msg.roomId, {
            roomId: msg.roomId,
            host: msg.host,
            gameType: msg.gameType,
            maxPlayers: msg.maxPlayers,
            roomName: msg.roomName,
            createdAt: Date.now()
          });
          break;

        case 'room-started':
        case 'room-closed':
          availableRooms.delete(msg.roomId);
          break;

        case 'room-updated':
          if (availableRooms.has(msg.roomId)) {
            Object.assign(availableRooms.get(msg.roomId), msg.updates);
          }
          break;
      }

      onLobbyUpdate(Array.from(availableRooms.values()));
    }
  });

  return {
    getRooms: () => Array.from(availableRooms.values()),
    cleanup: () => pubnub.unsubscribe({ channels: [lobbyChannel] })
  };
}
```

### Listing Rooms with Presence Counts

```javascript
async function getRoomPlayerCounts(pubnub, roomIds) {
  const channels = roomIds.map(id => `game.${id}`);

  const result = await pubnub.hereNow({
    channels,
    includeUUIDs: false
  });

  const counts = {};
  for (const [channel, data] of Object.entries(result.channels)) {
    const roomId = channel.replace('game.', '');
    counts[roomId] = data.occupancy;
  }

  return counts;
}
```

## Player Connection Management

### Presence Event Handling

```javascript
function setupPresenceHandling(pubnub, roomId, callbacks) {
  const roomChannel = `game.${roomId}`;

  pubnub.addListener({
    presence: (event) => {
      if (event.channel !== roomChannel) return;

      switch (event.action) {
        case 'join':
          callbacks.onPlayerJoin?.(event.uuid, event.timetoken);
          break;

        case 'leave':
          callbacks.onPlayerLeave?.(event.uuid, 'voluntary');
          break;

        case 'timeout':
          // Player lost connection
          callbacks.onPlayerDisconnect?.(event.uuid);
          break;

        case 'state-change':
          callbacks.onPlayerStateChange?.(event.uuid, event.state);
          break;
      }
    }
  });
}
```

### Handling Disconnections

```javascript
class PlayerConnectionManager {
  constructor(pubnub, roomId, options = {}) {
    this.pubnub = pubnub;
    this.roomId = roomId;
    this.reconnectTimeout = options.reconnectTimeout || 30000; // 30 seconds
    this.disconnectedPlayers = new Map();
  }

  handleDisconnect(playerId) {
    console.log(`Player ${playerId} disconnected`);

    // Start a reconnection timer
    const timer = setTimeout(() => {
      this.handleAbandon(playerId);
    }, this.reconnectTimeout);

    this.disconnectedPlayers.set(playerId, {
      disconnectedAt: Date.now(),
      timer
    });

    // Notify other players
    this.pubnub.publish({
      channel: `game.${this.roomId}`,
      message: {
        type: 'player-disconnected',
        playerId,
        reconnectDeadline: Date.now() + this.reconnectTimeout
      }
    });
  }

  handleReconnect(playerId) {
    const record = this.disconnectedPlayers.get(playerId);
    if (record) {
      clearTimeout(record.timer);
      this.disconnectedPlayers.delete(playerId);

      this.pubnub.publish({
        channel: `game.${this.roomId}`,
        message: {
          type: 'player-reconnected',
          playerId,
          timestamp: Date.now()
        }
      });
    }
  }

  handleAbandon(playerId) {
    this.disconnectedPlayers.delete(playerId);

    this.pubnub.publish({
      channel: `game.${this.roomId}`,
      message: {
        type: 'player-abandoned',
        playerId,
        timestamp: Date.now()
      }
    });
  }
}
```

## Error Handling

```javascript
pubnub.addListener({
  status: (statusEvent) => {
    switch (statusEvent.category) {
      case 'PNConnectedCategory':
        console.log('Connected to PubNub');
        break;

      case 'PNReconnectedCategory':
        console.log('Reconnected - requesting state recovery');
        requestStateSnapshot();
        break;

      case 'PNDisconnectedCategory':
        console.warn('Disconnected from PubNub');
        showReconnectingUI();
        break;

      case 'PNAccessDeniedCategory':
        console.error('Access denied - refresh auth token');
        refreshAuthToken();
        break;

      case 'PNNetworkIssuesCategory':
        console.warn('Network issues detected');
        break;
    }
  }
});

function requestStateSnapshot() {
  pubnub.publish({
    channel: `game.${currentRoomId}.state`,
    message: {
      type: 'state-request',
      requesterId: pubnub.getUserId(),
      timestamp: Date.now()
    }
  });
}
```

## Best Practices

1. **Use short presence timeouts** (15-20 seconds) for games so disconnected players are detected quickly; balance against false positives on flaky connections.

2. **Separate channels by concern** -- use distinct channels for game state, chat, and lobby traffic to avoid message congestion and simplify filtering.

3. **Store room metadata in App Context** rather than in-memory, so any client can look up room details without relying on a single source of truth.

4. **Always subscribe with `withPresence: true`** on game room channels to track player join/leave/timeout events automatically.

5. **Implement reconnection grace periods** -- give disconnected players 15-30 seconds to rejoin before treating them as abandoned.

6. **Generate unique, non-guessable room IDs** to prevent uninvited players from joining private games by guessing channel names.

7. **Clean up subscriptions** when leaving a room or closing the game; unsubscribe from all room-related channels to avoid unnecessary bandwidth.

8. **Use channel groups** when a player needs to subscribe to many channels simultaneously (e.g., multiple game lobbies), staying within the subscribe call limits.

9. **Test with realistic latency** -- PubNub delivers messages in ~30-100ms globally, but always test your game logic under simulated latency conditions.

10. **Rate limit publishing** on the client side to prevent flooding the game state channel; batch frequent updates into fewer messages where possible.
