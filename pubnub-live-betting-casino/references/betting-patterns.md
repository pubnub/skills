# PubNub Betting & Casino Patterns

## Overview

Casino table channels, in-play market suspension sequences, S1 deltas for blackjack/self-exclusion/geo, and tournament fan-out. Game rules, responsible-gambling product theory, and geo-compliance primers are out of scope.

## Casino table channels

| Game Type | State Model | Update Frequency | Channel Pattern |
|-----------|-------------|------------------|-----------------|
| Blackjack | Turn-based | Per action | `casino.blackjack.{tableId}` |
| Roulette | Round-based | Phase + result | `casino.roulette.{tableId}` |
| Slots | Instant | Spin result | `casino.slots.{userId}.{sessionId}` |
| Baccarat | Round-based | Deal, reveal, result | `casino.baccarat.{tableId}` |
| Poker | Turn-based | Per action, per street | `casino.poker.{tableId}` |
| Crash | Progressive | Multiplier ticks | `casino.crash.{roundId}` |

Actions publish on `{tableChannel}.action`. Table state broadcasts on the table channel after server validation.

### Blackjack Before-Publish (S1)

Use the canonical [server-authoritative Before-Publish pattern (S1)](../../pubnub-functions/references/functions-patterns.md#pattern-1-distributed-counter). **Blackjack-specific behavior:**

| Step | Blackjack delta |
|------|-----------------|
| Validate action | Reject if player not `active` or game not found |
| Mutate state | Apply hit / stand / double to player cards + total |
| Bust check | If `total > 21` after hit/double → `status = 'bust'` |
| Persist | `db.set(blackjack:{tableId}, gameState)` atomically before publish |
| Broadcast | `pubnub.publish` updated `game_state` on `casino.blackjack.{tableId}` |

Channel: `casino.blackjack.{tableId}.action` (On Publish). Deployable handler: start from [Pattern 1](../../pubnub-functions/references/functions-patterns.md#pattern-1-distributed-counter) and apply the table above.

Do not embed blackjack/roulette rule payloads or round-timing scripts here.

## In-play / live betting

### Event state

```javascript
async function publishEventState(pubnub, eventId, state) {
  await pubnub.publish({
    channel: `event.football.${eventId}`,
    message: {
      type: 'event_state', eventId,
      status: state.status,
      clock: state.clock, homeScore: state.homeScore, awayScore: state.awayScore,
      incidents: state.recentIncidents,
      timestamp: Date.now()
    }
  });
}
```

### Incident → market suspension sequence

```javascript
async function handleIncident(pubnub, eventId, incident) {
  const markets = await getEventMarkets(eventId);

  await Promise.all(markets.map(m =>
    pubnub.publish({
      channel: `event.football.${eventId}.market.${m.id}`,
      message: { type: 'market_suspension', marketId: m.id, suspended: true, reason: incident.type, timestamp: Date.now() }
    })
  ));

  await pubnub.publish({
    channel: `event.football.${eventId}`,
    message: { type: 'incident', incident, timestamp: Date.now() }
  });

  const newOdds = await recalculateOdds(eventId, incident);
  for (const m of markets) {
    if (m.shouldReopen) {
      await pubnub.publish({
        channel: `event.football.${eventId}.market.${m.id}`,
        message: { type: 'odds_update', marketId: m.id, selections: newOdds[m.id], suspended: false, timestamp: Date.now() }
      });
    }
  }
}
```

## Self-exclusion (S1)

Use the canonical [server-authoritative Before-Publish pattern (S1)](../../pubnub-functions/references/functions-patterns.md). **Self-exclusion delta:**

| Check | Rule |
|-------|------|
| KV key | `exclusion:{userId}` — read with `db.get` |
| Active guard | If `exclusion.active && Date.now() < exclusion.expiresAt` → abort |
| Abort payload | `{ type: 'access_denied', reason: 'self_excluded', expiresAt }` |
| Pass-through | If no exclusion record or expired → `request.ok()` |

Apply to any channel where wagers or access tokens are submitted. Deposit/session product theory lives with the operator, not in this skill.

## Geo-fence (S1)

Use [S1](../../pubnub-functions/references/functions-patterns.md) — XHR module for the geocoding call, KV Store for result caching. **Geo-fence delta:**

| Step | Betting-specific rule |
|------|-----------------------|
| Extract coords | `{ userId, latitude, longitude }` from `request.message` |
| XHR lookup | Reverse geocode to `countryCode` via external API (`xhr.fetch`) |
| Allowlist | Validate `countryCode` against jurisdiction-approved list (operator-configured) |
| KV cache | Store `{ allowed, country, verifiedAt, expiresAt }` under `geo:{userId}` — re-verify every 30 minutes |
| Abort | `{ type: 'geo_blocked', country }` if not allowed |

Counts **1 XHR op** per execution — cache geo result to avoid repeated lookups (see [S1 budget guidance](../../pubnub-functions/references/functions-patterns.md)).

## Multi-table tournament channels

| Channel Pattern | Purpose |
|----------------|---------|
| `tournament.{id}` | Tournament-wide announcements |
| `tournament.{id}.table.{tableId}` | Individual table game state |
| `tournament.{id}.leaderboard` | Live leaderboard updates |
| `tournament.{id}.lobby` | Pre-tournament lobby chat |

```javascript
async function joinTournament(pubnub, tournamentId, userId) {
  pubnub.subscribe({
    channels: [`tournament.${tournamentId}`, `tournament.${tournamentId}.lobby`, `tournament.${tournamentId}.leaderboard`]
  });
  await pubnub.publish({
    channel: `tournament.${tournamentId}.lobby`,
    message: { type: 'player_joined', userId, timestamp: Date.now() }
  });
}

async function updateLeaderboard(pubnub, tournamentId, standings) {
  await pubnub.publish({
    channel: `tournament.${tournamentId}.leaderboard`,
    message: {
      type: 'leaderboard_update',
      standings: standings.map((p, i) => ({ rank: i + 1, userId: p.userId, chips: p.chips, status: p.status })),
      timestamp: Date.now()
    }
  });
}
```

## Social bet sharing

```javascript
async function shareBet(pubnub, userId, bet) {
  await pubnub.publish({
    channel: 'social.feed',
    message: {
      type: 'shared_bet',
      sharedBy: userId,
      betId: bet.betId,
      selections: bet.selections.map(s => ({ selectionName: s.selectionName, odds: s.oddsAtSelection })),
      stake: bet.stake,
      timestamp: Date.now()
    }
  });
}
```

## Reconnect

On `PNReconnectedCategory`, refetch current table/game state and re-enable betting controls only after the snapshot arrives. Status wiring: [dropped-connections](../../pubnub-presence/references/dropped-connections.md).

## Best Practices

1. **Dedicated channels per table** so game state cannot leak across tables.
2. **Game logic in Functions (S1)** — server is the source of truth; link the owner, keep only the delta tables above.
3. **Sequence numbers** on game state and odds messages for out-of-order detection.
4. **Suspend markets before recalculation** during live incidents (sequence above).
5. **Self-exclusion and geo** as Before-Publish guards, not client-only checks.
6. **Presence** for seat occupancy on table channels.
7. **Encrypt** wager, balance, and cash-out channels.
8. **Hierarchical tournament channels** so players subscribe only to their table + lobby + leaderboard.
