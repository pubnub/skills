<!-- canonical-for: HISTORY_AND_PLAYBACK -->
<!-- canonical-for: TIMETOKEN_PAGINATION -->
<!-- used-by: pubnub-choose-docs-path, pubnub-keyset-management, pubnub-chat, pubnub-scale, pubnub-reliability -->

# History Pagination — Patterns and Pitfalls


## Timetokens (ordering key)

Timetokens are the canonical per-message ordering key. In JavaScript, treat them as **strings** (or BigInt) to avoid precision loss — retrieve conversion helpers from SDK docs.

## Ordering guarantees

- **Per-channel:** messages are ordered by timetoken.
- **Cross-channel:** no global ordering — merge client-side by timetoken if you need a unified timeline.

## Multi-channel fetch planning

When fetching multiple channels in one call, per-channel message caps are **lower** than single-channel mode. Plan pagination accordingly — retrieve current limits from SDK docs rather than hard-coding counts.

## Assembly with live traffic

| Scenario | Pattern |
|----------|---------|
| User reconnects mid-session | [Offline catch-up](offline-catch-up.md) + [dedup-on-merge](../../pubnub-reliability/references/dedup-on-merge.md) |
| Reactions/edits in history | Enable message actions in fetch params — retrieve current flag names from SDK docs |
| Unbounded backfill on every load | Cache last-seen timetoken; paginate incrementally |

## Anti-patterns

| Anti-pattern | Fix |
|--------------|-----|
| Single fetch assuming > platform page size | Paginate until short page |
| Multi-channel timeline without sort | Sort merged results by timetoken |
| `Date.now()` as start/end without conversion | Use SDK timetoken conversion |
| Full history replay on every page load | Persist cursor; fetch delta only |
| Live + history double-render | Dedup on merge at UI boundary |

## Orchestration

1. Confirm Message Persistence enabled on keyset ([retention-and-storage.md](retention-and-storage.md)).
2. Retrieve current `fetchMessages` API for the SDK language.
3. Implement cursor-based paging (oldest/newest timetoken cursors).
4. If subscribers are connected, merge live events with history using dedup rules.
