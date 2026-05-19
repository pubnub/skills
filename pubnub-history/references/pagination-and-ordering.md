<!-- canonical-for: HISTORY_AND_PLAYBACK -->
<!-- canonical-for: TIMETOKEN_PAGINATION -->
<!-- used-by: pubnub-choose-docs-path, pubnub-keyset-management, pubnub-chat, pubnub-scale, pubnub-reliability -->

# History: Pagination and Ordering

The canonical reference for the History API (`fetchMessages`), timetoken-based pagination, and per-channel ordering guarantees.

## Timetokens

A timetoken is a 17-digit integer (string in JSON to avoid precision loss in JavaScript) representing a moment in time at PubNub's resolution: **10 millionths of a second since Unix epoch**.

```
timetoken = unix_seconds * 10_000_000
```

To convert:

```javascript
const timetokenToDate = tt => new Date(Number(BigInt(tt) / 10000000n) * 1000);
const dateToTimetoken = d  => (BigInt(d.getTime()) * 10000n).toString();
```

Every published message receives a unique timetoken. Timetokens are the canonical ordering key.

## Basic `fetchMessages`

```javascript
const result = await pubnub.fetchMessages({
  channels: ['chat-room'],
  count: 100
});

const messages = result.channels['chat-room'] || [];
messages.forEach(msg => {
  console.log(msg.timetoken, msg.uuid, msg.message);
});
```

`count` defaults to 100 and is capped at 100 per request. Always specify it explicitly.

## Pagination by Timetoken

To page backward through history (older messages), use the `start` parameter with the oldest timetoken from the previous page.

```javascript
async function* iterateHistory(channel, opts = {}) {
  let start = opts.start;
  while (true) {
    const result = await pubnub.fetchMessages({
      channels: [channel],
      count: 100,
      start
    });
    const page = result.channels[channel] || [];
    if (page.length === 0) return;

    for (const msg of page) yield msg;

    if (page.length < 100) return;
    start = page[0].timetoken;
  }
}

for await (const msg of iterateHistory('chat-room')) {
  process(msg);
}
```

### Pagination Direction

| Param | Meaning |
|---|---|
| `start` | Fetch messages **older than** this timetoken (excludes the timetoken itself) |
| `end`   | Fetch messages **newer than** this timetoken (includes by default) |
| (both) | Bounded time range fetch |

```javascript
// Bounded fetch between two timetokens
const result = await pubnub.fetchMessages({
  channels: ['chat-room'],
  start: '17000000000000000',
  end:   '17100000000000000',
  count: 100
});
```

## Per-Channel Ordering Guarantee

PubNub guarantees per-channel ordering by timetoken. **There is no cross-channel ordering guarantee**.

Implications when fetching from multiple channels in one call:

```javascript
const result = await pubnub.fetchMessages({
  channels: ['ch-a', 'ch-b', 'ch-c'],
  count: 100
});

const merged = []
  .concat(result.channels['ch-a'] || [])
  .concat(result.channels['ch-b'] || [])
  .concat(result.channels['ch-c'] || []);

merged.sort((x, y) => BigInt(x.timetoken) < BigInt(y.timetoken) ? -1 : 1);
```

You **must** sort by timetoken yourself if you need a globally ordered stream across channels.

## Multi-Channel Fetch Limits

`fetchMessages` accepts up to **500 channels per call**, but with a much lower per-channel message limit when fetching multiple channels:

| Mode | Per-channel max |
|---|---|
| Single channel (`channels.length === 1`) | 100 |
| Multiple channels | **25 per channel** |

Plan for this: paginating across many channels means many calls.

## Including Message Actions and Metadata

```javascript
const result = await pubnub.fetchMessages({
  channels: ['chat-room'],
  count: 100,
  includeMessageActions: true,
  includeMeta: true
});
```

`includeMessageActions: true` is required to return reactions and edits along with each message. Note: this restricts you to single-channel fetch (the same call shape that allows up to 100 per page).

For Message Actions usage and semantics see [pubnub-chat/references/message-actions.md](../../pubnub-chat/references/message-actions.md).

## Including Message Type and `uuid`

The default response includes:

- `message` — the published payload
- `timetoken` — the unique ordering key
- `uuid` — the [publisher's userId](../../pubnub-app-developer/references/sdk-patterns.md)
- `messageType` — the [`type` parameter](../../pubnub-app-developer/references/publish-subscribe.md) if specified at publish time

## Common Patterns

### "Load Newer" / "Load Older" UI

```javascript
class HistoryPager {
  constructor(channel) {
    this.channel = channel;
    this.oldestTT = null;
    this.newestTT = null;
  }

  async loadInitial() {
    const r = await pubnub.fetchMessages({ channels: [this.channel], count: 50 });
    const page = r.channels[this.channel] || [];
    if (page.length) {
      this.oldestTT = page[0].timetoken;
      this.newestTT = page[page.length - 1].timetoken;
    }
    return page;
  }

  async loadOlder() {
    const r = await pubnub.fetchMessages({ channels: [this.channel], count: 50, start: this.oldestTT });
    const page = r.channels[this.channel] || [];
    if (page.length) this.oldestTT = page[0].timetoken;
    return page;
  }

  async loadNewer() {
    const r = await pubnub.fetchMessages({ channels: [this.channel], count: 50, end: this.newestTT });
    const page = r.channels[this.channel] || [];
    if (page.length) this.newestTT = page[page.length - 1].timetoken;
    return page;
  }
}
```

### Counting Messages in a Window (Without Reading All)

Use `messageCounts` (returns counts only, no payloads):

```javascript
const counts = await pubnub.messageCounts({
  channels: ['chat-room', 'announcements'],
  channelTimetokens: ['17000000000000000']
});
console.log(counts.channels);
```

## Common Anti-Patterns

| Anti-pattern | Fix |
|---|---|
| Single unbounded fetch with `count: 1000` | Cap is 100 — paginate |
| Assuming multi-channel fetch is globally ordered | Sort by timetoken client-side |
| Using `Date.now()` directly as start/end | Convert to timetoken (× 10000) first |
| Fetching the entire channel history on every page-load | Cache, persist last seen timetoken — see [offline-catch-up.md](offline-catch-up.md) |
| Reading history when live subscription is already running on the channel | Dedup on merge — see [pubnub-reliability/references/dedup-on-merge.md](../../pubnub-reliability/references/dedup-on-merge.md) |
| Multi-channel fetch expecting 100 per channel | Multi-channel mode caps at 25 per channel |

## Related Reading

- [offline-catch-up.md](offline-catch-up.md) — full reconnect flow
- [retention-and-storage.md](retention-and-storage.md) — what's available to fetch
- [pubnub-reliability/references/dedup-on-merge.md](../../pubnub-reliability/references/dedup-on-merge.md) — combining live + history
