<!-- canonical-for: OFFLINE_CATCH_UP -->
<!-- used-by: pubnub-choose-docs-path, pubnub-presence -->

# Offline Catch-up Flow

The canonical reference for catching a client up on missed messages after disconnect, with `restore`, last-seen-timetoken persistence, and merge-with-live semantics.

## The Two Catch-up Tiers

PubNub has two distinct mechanisms for delivering missed messages, and they have very different bounds:

| Tier | Mechanism | Window | Capacity |
|---|---|---|---|
| **Short-term cache** (always-on) | SDK auto-uses last timetoken on reconnect | ~2–20 minutes | ~100 messages per channel |
| **Message Persistence** (add-on) | Explicit `fetchMessages` from your code | retention period (1–unlimited days) | All stored messages |

**Brief network blip → short-term cache handles it automatically.**
**Long disconnect, app close-and-reopen, fresh device login → use Message Persistence.**

## SDK Auto Catch-up: Default Behavior

By default, the SDK stores the last received timetoken locally. On reconnect after a brief disconnect, it subscribes from that timetoken, and the server pushes anything in the cache that arrived since.

This works invisibly. **You don't need to write any code for it.** It's controlled by the `restore` option (see [SDK initialization patterns](../../pubnub-app-developer/references/sdk-patterns.md) for full `new PubNub(...)` config):

```javascript
const pubnub = new PubNub({
  subscribeKey: process.env.PN_SUBSCRIBE_KEY,
  userId: 'user-123',
  restore: true
});
```

## Disabling Auto Catch-up: `restore: false`

When you want subscribers to receive *only* messages from "now" forward (no replay of cached messages), set `restore: false`. Common cases:

- Live ticker / scoreboard where stale messages would confuse the UI
- A channel with a very high publish rate where the cached burst would be overwhelming
- Apps that fetch their own historical view independently

```javascript
const pubnub = new PubNub({
  subscribeKey: process.env.PN_SUBSCRIBE_KEY,
  userId: 'user-123',
  restore: false   // start fresh on every (re)connect
});
```

`restore: false` does **not** disable the History API. You can still fetch missed messages explicitly via `fetchMessages` if Message Persistence is enabled.

## Full Offline Catch-up Pattern (Message Persistence)

For app close-and-reopen, fresh device login, or disconnects beyond the short-term cache window, use this pattern:

### 1. Persist Last-Seen Timetoken

Each time you process a live message, persist its timetoken in local storage / IndexedDB / a server-side store keyed to the user.

```javascript
function processMessage(msg) {
  renderMessage(msg);
  localStorage.setItem(`lastTT:${channel}`, msg.timetoken);
}

pubnub.addListener({
  message: e => processMessage(e)
});
```

### 2. On Reconnect / App Open: Fetch From Last Seen

```javascript
async function catchUp(channel) {
  const lastTT = localStorage.getItem(`lastTT:${channel}`);
  if (!lastTT) return [];   // first run — nothing to catch up

  const all = [];
  let endCursor = null;

  while (true) {
    const r = await pubnub.fetchMessages({
      channels: [channel],
      count: 100,
      start: endCursor || undefined,
      end:   lastTT
    });
    const page = r.channels[channel] || [];
    if (!page.length) break;

    all.push(...page);
    endCursor = page[0].timetoken;

    if (page.length < 100) break;
  }

  return all.sort((a, b) => BigInt(a.timetoken) < BigInt(b.timetoken) ? -1 : 1);
}
```

For pagination details see [pagination-and-ordering.md](pagination-and-ordering.md).

### 3. Subscribe (Optionally Skip the Auto Catch-up Cache)

For full catch-up patterns, set `restore: false` on the PubNub instance — your explicit `fetchMessages` is the source of truth, not the short-term cache.

For [`pubnub.subscribe` and `addListener` mechanics](../../pubnub-app-developer/references/publish-subscribe.md) see the canonical owner.

```javascript
pubnub.subscribe({ channels: [channel] });
```

### 4. Merge Caught-up + Live, Dedup, Render

Now both `fetchMessages` (caught-up) and the live subscription are delivering messages. There may be overlap. **Dedup is mandatory** — see [pubnub-reliability/references/dedup-on-merge.md](../../pubnub-reliability/references/dedup-on-merge.md).

```javascript
const seenTTs = new Set();
function process(msg) {
  if (seenTTs.has(msg.timetoken)) return;   // dedup
  seenTTs.add(msg.timetoken);
  renderMessage(msg);
  localStorage.setItem(`lastTT:${channel}`, msg.timetoken);
}

const caughtUp = await catchUp(channel);
caughtUp.forEach(process);

pubnub.addListener({
  message: e => process(e)
});
pubnub.subscribe({ channels: [channel] });
```

### 5. Bound the Catch-up Window

Don't unconditionally catch up if the user has been offline for days. Cap the window:

```javascript
const ONE_DAY_TT = 24n * 60n * 60n * 10000000n;
const nowTT = (BigInt(Date.now()) * 10000n).toString();

let lastTT = localStorage.getItem(`lastTT:${channel}`);
if (lastTT && BigInt(nowTT) - BigInt(lastTT) > ONE_DAY_TT) {
  lastTT = (BigInt(nowTT) - ONE_DAY_TT).toString();   // cap to 1 day
}
```

This protects against the user-was-offline-for-a-month case where catching up the whole interval is pointless and expensive.

## Connection State Listener

Use the SDK's connection-state event to drive catch-up only on actual reconnects, not on first connect:

```javascript
pubnub.addListener({
  status: async (s) => {
    if (s.category === 'PNReconnectedCategory') {
      const missed = await catchUp(channel);
      missed.forEach(process);
    }
  }
});
```

For the full reconnect-and-backoff pattern see [pubnub-reliability/references/backoff-and-jitter.md](../../pubnub-reliability/references/backoff-and-jitter.md).

## Application-Level Stale Filter

Even with `restore: true` and short-term cache catch-up, you might want to drop messages that are too stale to display:

```javascript
const STALE_MS = 60_000;
function shouldDisplay(msg) {
  const ageMs = Date.now() - Number(BigInt(msg.timetoken) / 10000n);
  return ageMs < STALE_MS;
}
```

This is application-level: useful for live tickers, presence indicators, or anything where displaying stale data is worse than dropping.

## Common Anti-Patterns

| Anti-pattern | Fix |
|---|---|
| Catching up unbounded — replays days of messages on a quick blip | Cap to a sensible window (1 hour, 1 day) |
| Forgetting to dedup live + caught-up | Use a `Set` of timetokens; see [pubnub-reliability/references/dedup-on-merge.md](../../pubnub-reliability/references/dedup-on-merge.md) |
| Fetching from time-zero on first run | First run has no last-seen — render an "earliest available" page or skip catch-up |
| Persisting last-seen as a `Date` instead of timetoken | Timetokens are unique-per-message; dates collide and round-trip lossy |
| Running catch-up on `PNConnectedCategory` (initial connect) | Only on `PNReconnectedCategory` |
| Using `restore: false` and forgetting to fetch missed messages | Decide explicitly — either auto catch-up or explicit fetch, but the user-visible result must be complete |

## Related Reading

- [pagination-and-ordering.md](pagination-and-ordering.md) — `fetchMessages` mechanics
- [pubnub-reliability/references/dedup-on-merge.md](../../pubnub-reliability/references/dedup-on-merge.md) — dedup live + history
- [pubnub-reliability/references/backoff-and-jitter.md](../../pubnub-reliability/references/backoff-and-jitter.md) — reconnect-with-backoff before catch-up
- [pubnub-presence/references/presence-events.md](../../pubnub-presence/references/presence-events.md) — `PNReconnectedCategory` and connection state
