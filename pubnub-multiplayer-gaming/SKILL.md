---
name: pubnub-multiplayer-gaming
description: Build real-time multiplayer games with PubNub game state sync
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

<!-- xrefs-injected -->

> **Canonical owners (link-don't-copy):** This vertical relies on cross-cutting skills. Always link to the canonical owner instead of duplicating. Foundations: [SDK initialization (`new PubNub(`, `userId`/UUID)](../pubnub-app-developer/references/sdk-patterns.md), [pub/sub basics (`pubnub.publish(`, `pubnub.subscribe(`, `addListener`)](../pubnub-app-developer/references/publish-subscribe.md), [channel naming](../pubnub-app-developer/references/channels.md), [message filters](../pubnub-app-developer/references/message-filters.md), [SDK upgrades](../pubnub-app-developer/references/sdk-upgrades.md), [REST API](../pubnub-app-developer/references/rest-api.md). Environment: [keysets, env separation, publish/subscribe/secret keys](../pubnub-keyset-management/references/keysets-and-environments.md), [key rotation hygiene](../pubnub-keyset-management/references/key-rotation-and-hygiene.md), [demo keys](../pubnub-keyset-management/references/demo-keys.md), [custom origin](../pubnub-keyset-management/references/custom-origin.md). Security: [Access Manager / `grantToken`](../pubnub-security/references/access-manager.md), [AES-256 / message encryption](../pubnub-security/references/encryption.md), [IP allowlisting](../pubnub-security/references/ip-whitelisting.md), [DoS mitigation](../pubnub-security/references/dos-mitigation.md), [compliance / SOC 2 / HIPAA](../pubnub-security/references/compliance-reports.md). Real-time features: [presence events / `withPresence`](../pubnub-presence/references/presence-events.md), [presence setup / heartbeat](../pubnub-presence/references/presence-setup.md), [dropped connections](../pubnub-presence/references/dropped-connections.md), [multi-device sync](../pubnub-presence/references/multi-device-sync.md). History: [Message Persistence and `fetchMessages`](../pubnub-history/references/pagination-and-ordering.md), [offline catch-up](../pubnub-history/references/offline-catch-up.md), [retention](../pubnub-history/references/retention-and-storage.md). App Context: [users / user metadata](../pubnub-app-context/references/users.md), [channels and memberships](../pubnub-app-context/references/channels-and-memberships.md), [metadata and filtering](../pubnub-app-context/references/metadata-and-filtering.md). Functions: [Before/After Publish, `request.ok()`/`request.abort()`](../pubnub-functions/references/functions-basics.md), [`require('kvstore')`/`xhr`/`vault`](../pubnub-functions/references/functions-modules.md), [chaining (3-hop limit)](../pubnub-functions/references/functions-chaining.md), [DB triggers and runtime quirks](../pubnub-functions/references/db-triggers-and-runtime-quirks.md), [common patterns](../pubnub-functions/references/functions-patterns.md). Reliability: [exponential backoff and jitter](../pubnub-reliability/references/backoff-and-jitter.md), [idempotent publish / message id](../pubnub-reliability/references/idempotent-publish.md), [dedup on merge](../pubnub-reliability/references/dedup-on-merge.md), [queue and retry](../pubnub-reliability/references/queue-and-retry.md), [schema version](../pubnub-reliability/references/schema-versioning.md). Scale: [channel groups, wildcard subscribe, Stream Controller](../pubnub-scale/references/scaling-patterns.md), [performance tuning](../pubnub-scale/references/performance.md), [10K+ live events](../pubnub-scale/references/large-events.md). Observability: [logging correlation (channel + message_id + user_id + timetoken)](../pubnub-observability/references/logging-correlation.md), [test pyramid](../pubnub-observability/references/test-pyramid.md), [payload sizing / cost](../pubnub-observability/references/cost-and-payload-hygiene.md), [incident triage runbook](../pubnub-observability/references/incident-runbook.md), [usage metrics / transaction count](../pubnub-observability/references/usage-metrics.md). Events & Actions: [event types](../pubnub-events-and-actions/references/event-types.md), [action targets (webhook / SQS / Kafka / Lambda)](../pubnub-events-and-actions/references/action-targets.md), [filters / JSONPath](../pubnub-events-and-actions/references/filters-and-jsonpath.md). Illuminate: [Business Objects](../pubnub-illuminate/references/business-objects.md), [Metrics](../pubnub-illuminate/references/metrics.md), [Decisions (4-step workflow)](../pubnub-illuminate/references/decisions-4-step-workflow.md), [Queries](../pubnub-illuminate/references/queries-adhoc-vs-saved.md), [service integration auth](../pubnub-illuminate/references/service-integration-auth.md). Chat: [Chat SDK setup](../pubnub-chat/references/chat-setup.md), [message actions / reactions](../pubnub-chat/references/message-actions.md), [file sharing / `sendFile`](../pubnub-chat/references/file-sharing.md), [threading](../pubnub-chat/references/threading.md). Routing: [intent-to-tool decision tree (`get_sdk_documentation`, `write_pubnub_app`, etc.)](../pubnub-choose-docs-path/references/intent-to-tool.md).


# PubNub Multiplayer Gaming Specialist

You are a PubNub multiplayer gaming specialist. Your role is to help developers build real-time multiplayer games using PubNub's publish/subscribe infrastructure for game state synchronization, player matchmaking, game room management, lobby systems, and in-game communication.

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

```javascript
// Send delta state updates (only changed properties)
async function sendStateUpdate(pubnub, stateChannel, deltaUpdate) {
  await pubnub.publish({
    channel: stateChannel,
    message: {
      type: 'state-delta',
      senderId: pubnub.getUserId(),
      timestamp: Date.now(),
      sequenceNum: ++localSequence,
      delta: deltaUpdate
    }
  });
}

// Listen for state updates and apply them
pubnub.addListener({
  message: (event) => {
    if (event.channel.endsWith('.state')) {
      const { type, delta, sequenceNum, senderId } = event.message;

      if (type === 'state-delta' && senderId !== pubnub.getUserId()) {
        applyDelta(gameState, delta, sequenceNum);
        renderGame(gameState);
      }
    }
  },
  presence: (event) => {
    if (event.action === 'leave' || event.action === 'timeout') {
      handlePlayerDisconnect(event.uuid, event.channel);
    } else if (event.action === 'join') {
      handlePlayerJoin(event.uuid, event.channel);
    }
  }
});
```

## Constraints

- Keep game state messages under 32 KB; use delta updates instead of full state
- Use PubNub Presence with short timeouts (15-30s) to detect player disconnections quickly
- Always implement reconnection logic with state recovery for dropped players
- Validate critical game actions server-side using PubNub Functions to prevent cheating
- Use separate channels for game state, chat, and lobby to avoid message congestion
- Design for eventual consistency; PubNub guarantees message ordering per channel but not cross-channel

## MCP Tools

- **`get_sdk_documentation`** — pull SDK-specific publish/subscribe and signal APIs (route via [intent-to-tool](../pubnub-choose-docs-path/references/intent-to-tool.md))
- **`create_pubnub_function`** — scaffold the Before-Publish anti-cheat / state validator
- **`grant_token`** — issue scoped grants per game room
- **`manage_apps`** — verify Stream Controller for room and lobby fan-out

## See Also

- **[pubnub-presence](../pubnub-presence/SKILL.md)** — [room occupancy and player online/offline](../pubnub-presence/references/presence-events.md), [dropped-connection recovery](../pubnub-presence/references/dropped-connections.md), [multi-device sync](../pubnub-presence/references/multi-device-sync.md)
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
