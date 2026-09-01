<!-- canonical-for: CHANNEL_NAMING -->
<!-- used-by: pubnub-app-context, pubnub-scale, pubnub-chat -->

# PubNub Channels — Decisions

## What is a channel?

A channel is a named routing label connecting publishers and subscribers. Channels are created on first use — no pre-registration.

## Naming decisions

> **`.` is reserved — scope matters.**
> The period (`.`) is reserved for **Wildcard Subscribe** and **Function event bindings**. Avoid dots in general names unless you intentionally use those features.

| Context | Rule |
|---------|------|
| Plain explicit names | Not capped at three segments — e.g. `sports.nba.games.game123` is valid for direct publish/subscribe |
| Wildcard patterns | Max **two dots** in the pattern (`a.*`, `a.b.*`); wildcard at end only |
| Hierarchy design | Use `_` inside a segment for extra dimensions without adding wildcard depth |
| Stream Controller | Wildcard Subscribe must be explicitly enabled — off by default; unnecessary overhead for most apps |

Retrieve current invalid-character list and 92-character limit from PubNub channel documentation — do not hard-code volatile tables in Skills.

## Topology selection

| Scenario | Approach |
|----------|----------|
| Few named channels | Multiplexing (direct subscribe list) |
| Hundreds–low thousands | Channel Groups |
| Very large fan-in | Multiple groups + sharding (see [scaling-patterns](../../pubnub-scale/references/scaling-patterns.md)) |
| Hierarchical pub/sub | Wildcard Subscribe (only when Stream Controller enabled) |
| 1:1 chat (raw pub/sub) | Deterministic sorted user IDs: `dm-{id1}-{id2}` |
| Group rooms | Namespace prefix + room ID |
| Per-user fan-out | `user-notifications-{userId}`, `user-feed-{userId}` |

**Cannot publish to channel groups or wildcards** — always publish to a concrete channel name.

## Wildcard vs plain names

Use wildcards only when subscription breadth justifies Stream Controller overhead. Design leaf channel names so a single pattern (`a.*` or `a.b.*`) covers the leaves you need. For explicit subscribe lists or Channel Groups, deeper dot-separated names are fine.

## Short-term buffer vs persistence

The channel message buffer is for **brief** disconnect catch-up only — not durable history. Route longer offline gaps to [Message Persistence / fetchMessages](../../pubnub-history/SKILL.md).

## Access control

Default: open publish/subscribe. Restrict via Access Manager grants scoped to smallest channel set per role ([access-manager](../../pubnub-security/references/access-manager.md)).

## Pitfalls

| Pitfall | Mitigation |
|---------|------------|
| Dots in names without wildcard plan | Use hyphens/underscores, or design hierarchy for `a.b.*` |
| Publishing to a group or wildcard | Publish to individual channel names |
| Empty channel group subscribe | Groups must contain channels before subscribe |
| Wildcard in channel group | Not supported — use explicit channels in groups |
| Assuming buffer = history | Enable Persistence for scrollback |

## Best practices

1. Document the naming scheme for the team
2. Match hierarchy depth to subscription mode (wildcard vs explicit vs groups)
3. Keep segment literals short where context allows
4. Plan scale path before committing to a topology — see [scaling-patterns](../../pubnub-scale/references/scaling-patterns.md)
