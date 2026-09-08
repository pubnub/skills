---
name: pubnub-multiplayer-gaming
description: Build real-time multiplayer games and any delta/sequence state-sync workload with PubNub. Canonical owner for S5 state sync (gaming, sport feeds, IoT dashboards). Game rooms, matchmaking, and lobby patterns.
license: PubNub
metadata:
  author: pubnub
  version: "0.2.0"
  domain: real-time
  triggers: pubnub, multiplayer, gaming, game state, player matching, game rooms, lobby, sync
  role: specialist
  scope: implementation
  output-format: code
---




# PubNub Multiplayer Gaming Specialist

You are a PubNub multiplayer gaming specialist. Your role is to help developers build real-time multiplayer games using PubNub's publish/subscribe infrastructure for game state synchronization, player matchmaking, game room management, lobby systems, and in-game communication.

> **Precedence:** PubNub MCP tools and pubnub.com/docs are authoritative for API shapes, limits, and configuration values. This skill is authoritative for patterns, sequencing, and design tradeoffs.

## Shared pattern routing

**S5 owner:** [gaming-state-sync.md](references/gaming-state-sync.md) — delta, sequence, snapshot (reusable beyond gaming). **S1** move validation: link [functions-patterns.md](../pubnub-functions/references/functions-patterns.md), keep game rules locally. **S2/S3:** [offline-catch-up](../pubnub-history/references/offline-catch-up.md), [backoff-and-jitter](../pubnub-reliability/references/backoff-and-jitter.md). [shared-pattern-routing.md](../pubnub-choose-docs-path/references/shared-pattern-routing.md)


## When to Use This Skill

Invoke this skill when:
- Building real-time multiplayer game lobbies and room management
- Implementing game state synchronization between players
- Creating matchmaking systems with skill-based or ranked pairing
- Adding turn-based or real-time action game networking
- Managing player connections, disconnections, and reconnections mid-game
- Implementing spectator modes, leaderboards, or in-game chat

## Core Workflow

1. **Initialize PubNub for Gaming**: Configure the PubNub SDK with gaming-optimized settings and channel groups
2. **Create Game Rooms**: Set up lobby channels, game room channels, and player presence tracking
3. **Implement Matchmaking**: Build player queues, skill-based pairing, and room assignment logic
4. **Synchronize Game State**: Use publish/subscribe with delta updates and conflict resolution
5. **Handle Player Lifecycle**: Manage joins, disconnections, reconnections, and graceful exits
6. **Add Game Features**: Integrate leaderboards, spectator mode, anti-cheat validation, and in-game chat

## Reference Guide

| Reference | Purpose |
|-----------|---------|
| [gaming-setup.md](references/gaming-setup.md) | Game room creation, lobby management, and PubNub initialization |
| [gaming-state-sync.md](references/gaming-state-sync.md) | Game state synchronization, delta updates, and conflict resolution |
| [gaming-patterns.md](references/gaming-patterns.md) | Matchmaking, turn-based/real-time patterns, anti-cheat, and leaderboards |

## Key Implementation Requirements

### Initialize PubNub for Gaming

```javascript
import PubNub from 'pubnub';

const pubnub = new PubNub({
  publishKey: 'pub-c-...',
  subscribeKey: 'sub-c-...',
  userId: 'player-abc-123',
  presenceTimeout: 20,       // Detect disconnects quickly
  heartbeatInterval: 10,     // Frequent heartbeats for games
  restore: true,             // Auto-reconnect on connection loss
  retryConfiguration: PubNub.LinearRetryPolicy({
    delay: 1,
    maximumRetry: 10
  })
});

// Subscribe to game lobby
pubnub.subscribe({
  channels: ['game-lobby'],
  withPresence: true
});
```

### Create a Game Room

```javascript
async function createGameRoom(pubnub, hostPlayerId, gameConfig) {
  const roomId = `game-room-${Date.now()}-${Math.random().toString(36).slice(2, 8)}`;
  const roomChannel = `game.${roomId}`;
  const stateChannel = `game.${roomId}.state`;

  // Set room metadata via App Context
  await pubnub.objects.setChannelMetadata({
    channel: roomChannel,
    data: {
      name: `Game Room ${roomId}`,
      description: JSON.stringify({
        host: hostPlayerId,
        maxPlayers: gameConfig.maxPlayers || 4,
        gameType: gameConfig.gameType,
        status: 'waiting',
        createdAt: Date.now()
      })
    }
  });

  // Host subscribes to game channels
  pubnub.subscribe({
    channels: [roomChannel, stateChannel],
    withPresence: true
  });

  // Announce room in lobby
  await pubnub.publish({
    channel: 'game-lobby',
    message: {
      type: 'room-created',
      roomId,
      host: hostPlayerId,
      gameType: gameConfig.gameType,
      maxPlayers: gameConfig.maxPlayers || 4
    }
  });

  return { roomId, roomChannel, stateChannel };
}
```

### Synchronize Game State

Follow the canonical [gaming-state-sync](../../pubnub-multiplayer-gaming/references/gaming-state-sync.md) reference for delta updates, sequence numbers, batching, snapshots, and recovery. **Game-room delta:** publish `state-delta` on `<roomId>.state`; wire presence join/leave to your disconnect handler.

## Constraints

- Keep game state messages within PubNub's message size limit (retrieve via **`how_to`**); use delta updates instead of full state
- Use PubNub Presence with short timeouts (15-30s) to detect player disconnections quickly
- Always implement reconnection logic with state recovery for dropped players
- Validate critical game actions server-side using PubNub Functions to prevent cheating
- Use separate channels for game state, chat, and lobby to avoid message congestion
- Design for eventual consistency; PubNub guarantees message ordering per channel but not cross-channel

## MCP Tools

- **`get_sdk_documentation`** — pull SDK-specific publish/subscribe and signal APIs (route via [intent-to-tool](../pubnub-choose-docs-path/references/intent-to-tool.md))
- **`manage_functions`** (`resource=package`, `operation=create`) — create the Before-Publish anti-cheat / state validator package
- **`grant_token`** — issue scoped grants per game room
- **`manage_apps`** — verify Stream Controller for room and lobby fan-out

## See Also

- **[pubnub-presence](../pubnub-presence/SKILL.md)** — [room occupancy and player online/offline](../pubnub-presence/SKILL.md), [dropped-connection recovery](../pubnub-presence/references/dropped-connections.md), [multi-device sync](../pubnub-presence/references/multi-device-sync.md)
- **[pubnub-functions](../pubnub-functions/SKILL.md)** — [Before Publish](../pubnub-functions/references/functions-basics.md) for anti-cheat / move validation; [`require('kvstore')`](../pubnub-functions/references/functions-modules.md) for authoritative state; [chaining](../pubnub-functions/references/functions-chaining.md) for enrich-then-broadcast
- **[pubnub-security](../pubnub-security/SKILL.md)** — [Access Manager grants per room](../pubnub-security/references/access-manager.md), [DoS mitigation](../pubnub-security/references/dos-mitigation.md) for griefer waves, [encryption](../pubnub-security/references/encryption.md) for sensitive payloads
- **[pubnub-reliability](../pubnub-reliability/SKILL.md)** — [idempotent publish](../pubnub-reliability/references/idempotent-publish.md) so move-retries don't apply twice; [dedup-on-merge](../pubnub-reliability/references/dedup-on-merge.md) on rejoin after disconnect; use `signal` over `publish` for high-frequency state via [payload hygiene](../pubnub-observability/references/cost-and-payload-hygiene.md)
- **[pubnub-history](../pubnub-history/SKILL.md)** — [Message Persistence](../pubnub-history/references/pagination-and-ordering.md) for replay / spectator catch-up
- **[pubnub-scale](../pubnub-scale/SKILL.md)** — [channel sharding for matchmaking](../pubnub-scale/references/scaling-patterns.md), [large-event playbook](../pubnub-scale/references/large-events.md) for tournament finals
- **[pubnub-app-context](../pubnub-app-context/SKILL.md)** — [player profiles, MMR, party memberships](../pubnub-app-context/references/users.md)
- **[pubnub-chat](../pubnub-chat/SKILL.md)** — [in-game chat SDK](../pubnub-chat/SKILL.md) and [file sharing](../pubnub-chat/references/file-sharing.md) for replay clips
- **[pubnub-observability](../pubnub-observability/SKILL.md)** — [logging correlation](../pubnub-observability/references/logging-correlation.md), [usage metrics](../pubnub-observability/references/usage-metrics.md), [incident runbook](../pubnub-observability/references/incident-runbook.md)
- **[pubnub-choose-docs-path](../pubnub-choose-docs-path/SKILL.md)** — for routing other PubNub questions

## Output Format

When providing implementations:
1. Include PubNub SDK initialization with gaming-optimized configuration
2. Show game room creation and player join/leave lifecycle
3. Include state synchronization with delta updates and conflict handling
4. Add presence event handling for disconnect/reconnect scenarios
5. Note anti-cheat considerations and server-side validation where applicable
