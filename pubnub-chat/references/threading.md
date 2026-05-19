<!-- canonical-for: THREADING -->
<!-- used-by: -->

> **Cross-references:** Threads are implemented as separate channels under the hood — see [channel naming](../../pubnub-app-developer/references/channels.md). Built on [pub/sub basics](../../pubnub-app-developer/references/publish-subscribe.md). Thread channels carry their own [App Context channel metadata and memberships](../../pubnub-app-context/references/channels-and-memberships.md) (see also [App Context overview](../../pubnub-app-context/references/users.md)). Reactions inside threads use [Message Actions](message-actions.md). For [Message Persistence retention so threads can be scrolled back](../../pubnub-history/references/pagination-and-ordering.md) and [retention configuration](../../pubnub-history/references/retention-and-storage.md) see `pubnub-history`.

# Threaded Messages

A thread is a side conversation attached to a parent message. In PubNub Chat SDK, each thread is implemented as its own channel, deterministically named from the parent message's timetoken.

## Concepts

| Concept | Definition |
|---|---|
| **Parent message** | The message a thread hangs off of |
| **Thread channel** | A child channel that holds the thread's messages |
| **Thread root** | First reply in the thread |
| **Thread preview** | Lightweight summary attached to the parent (count, last reply) |

## Create a Thread

```javascript
const message = /* a Message instance */;
const thread = await message.createThread();
```

## Reply in a Thread

```javascript
await thread.sendText('Reply in thread');
```

Same API as a regular channel — under the hood it's just publishing to the thread channel.

## Connect to a Thread

```javascript
thread.connect((reply) => {
  console.log('Thread reply:', reply.text);
});
```

Treat the thread like any other channel. Subscribe / unsubscribe / fetch history identically.

## Thread History

```javascript
const { messages } = await thread.getHistory({ count: 50 });
```

Backed by the same [Message Persistence retention](../../pubnub-history/references/retention-and-storage.md) as the parent channel.

## Thread Previews

The Chat SDK maintains a counter-style preview on the parent message. Read it via:

```javascript
const replyCount = await message.getThread();
```

## Constraints

- **Threads are channels** — every thread you create counts as a channel for billing and presence purposes (see [usage metrics owner](../../pubnub-observability/references/usage-metrics.md)).
- **No nested threads** — you cannot create a thread inside a thread reply. Design the UI accordingly.
- **Thread channel name is deterministic** — `PUBNUB_INTERNAL_THREAD.<parentChannel>.<parentTimetoken>`. Don't write to that channel name directly.
- **Permissions** — grant Access Manager access to the thread channel separately from the parent (see [Access Manager](../../pubnub-security/references/access-manager.md)).
- **Cleanup** — deleting the parent message does **not** delete the thread channel. Add `chat.removeThreadChannel(thread)` if needed.

## Common Pitfalls

| Pitfall | Mitigation |
|---|---|
| Forgot to grant access to thread channel | Re-grant on the deterministic thread channel name |
| Thread previews stale on UI | Subscribe to thread parent's Message Actions to see preview updates ([message-actions.md](message-actions.md)) |
| Many small empty threads pollute billing | Lazy-create threads only on first reply |
| Thread channels never cleaned up | Run a periodic GC of empty thread channels |
| Wanted nested threads | Not supported; flatten replies or use `quotedMessage` instead |

## Quote vs Thread

If you want "reply with quote" semantics in the same channel rather than a side conversation, use the Chat SDK's `sendText({ quotedMessage })` — it stays in the parent channel and doesn't create a new channel.
