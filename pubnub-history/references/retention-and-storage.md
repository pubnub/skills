<!-- canonical-for: HISTORY_RETENTION -->
<!-- used-by: -->

# History: Retention and Selective Storage

The canonical reference for configuring message retention, opting messages out of storage, and treating History as a catch-up window rather than a primary data store.

## Enabling Message Persistence

1. Log in to the [Admin Portal](https://admin.pubnub.com).
2. Open the keyset.
3. Enable **Message Persistence** add-on.
4. Choose a **Retention Period**.

For environment-separated configuration (different retention per env), see [pubnub-keyset-management/references/keysets-and-environments.md](../../pubnub-keyset-management/references/keysets-and-environments.md).

## Retention by Plan

| Plan | Available retention |
|---|---|
| Free | Up to 7 days |
| Starter | Up to 180 days |
| Pro | Up to unlimited |

Choose the shortest period that satisfies your catch-up window. Longer retention costs more.

### When to Pick Which

| Use case | Suggested retention |
|---|---|
| Live ticker, presence updates | 1–7 days (catch-up only) |
| Chat with reasonable session continuity | 30 days |
| Chat needing weeks-long history scrollback | 180 days |
| Compliance archival | Unlimited (Pro) **OR** export to your own data store |
| Telemetry / IoT | Forward to your warehouse; keep PubNub retention short |

## Selective Storage

Even with the add-on enabled, you can opt individual messages out of storage with `storeInHistory: false`:

```javascript
await pubnub.publish({
  channel: 'chat-room',
  message: { typingNotification: true, userId: 'user-123' },   // see [userId rules](../../pubnub-app-developer/references/sdk-patterns.md)
  storeInHistory: false
});
```

Common candidates for `storeInHistory: false`:

- Typing indicators
- Presence-like ephemeral signals
- Cursor position updates
- Game state ticks (deltas) where only the latest matters
- Anything you'd never want to replay on reconnect

`storeInHistory: false` messages are **not retrievable**. There is no fallback fetch; the data is delivered live and then gone.

For ephemeral, non-stored messages, also consider `signal()` instead of `publish()` — see [pubnub-app-developer/references/publish-subscribe.md](../../pubnub-app-developer/references/publish-subscribe.md).

## Message Size and Cost

Larger payloads consume more storage and bandwidth. Keep payloads small:

- Strip server-side fields a client doesn't need
- Send IDs + small refs instead of large embedded blobs
- For files, use the [File Sharing API](../../pubnub-chat/references/file-sharing.md), not `publish()`

For [payload sizing](../../pubnub-observability/references/cost-and-payload-hygiene.md) best practices in general see also [pubnub-scale/references/scaling-patterns.md](../../pubnub-scale/references/scaling-patterns.md).

## Catch-up Tool, Not a Data Lake

Message Persistence is optimized for:

- Hours-to-days catch-up window
- Per-channel time-ordered fetch
- Reasonable per-channel volumes

It is **not** optimized for:

- Long-term BI / analytics
- Full-text search
- Compliance archival of years of history
- Cross-channel aggregation

For those use cases, forward messages to your own data store via:

- A [PubNub Function](../../pubnub-functions/SKILL.md) on `publish` events that writes to your DB
- An [Events & Actions](../../pubnub-events-and-actions/SKILL.md) integration to Lambda, Kafka, S3, or BigQuery
- A subscriber service that bulk-writes to your warehouse

## Verifying Storage in Lower Environments

To verify a publish actually stored:

```javascript
const result = await pubnub.publish({ channel: 'test', message: { hello: 'world' } });
console.log('Published timetoken:', result.timetoken);

await new Promise(r => setTimeout(r, 1000));   // give storage time to flush

const fetched = await pubnub.fetchMessages({ channels: ['test'], count: 10 });
console.log('Fetched:', fetched.channels['test']);
```

In dev, use a short retention (1 day) so test data automatically ages out.

## Cleanup and Deletion

To delete stored messages within a time range (privacy / GDPR / right-to-be-forgotten):

```javascript
await pubnub.deleteMessages({
  channel: 'chat-room',
  start: '17000000000000000',   // delete messages newer than this
  end:   '17100000000000000'    // and older than this
});
```

`deleteMessages` requires:

- The keyset to have **Delete from History** enabled in the Admin Portal
- The PubNub instance initialized with the [secret key](../../pubnub-keyset-management/references/keysets-and-environments.md) (server-only operation)

For production deletion at scale, do it from a server, not a client.

## Common Anti-Patterns

| Anti-pattern | Fix |
|---|---|
| Storing typing indicators in History | Use `storeInHistory: false` or `signal()` |
| Long retention as a substitute for analytics | Forward to your own data store |
| Deleting from a client | Server-side only (requires secret key) |
| Same retention across all environments | Use shorter retention in dev/staging |
| Treating PubNub History as searchable | It's not full-text searchable; forward to a search index if needed |

## Related Reading

- [pagination-and-ordering.md](pagination-and-ordering.md) — fetching what's stored
- [offline-catch-up.md](offline-catch-up.md) — using stored messages for resume
- [pubnub-keyset-management/references/keysets-and-environments.md](../../pubnub-keyset-management/references/keysets-and-environments.md) — enabling the add-on per env
