<!-- canonical-for: MESSAGE_ACTIONS -->
<!-- used-by: -->

> **Cross-references:** Built on [pub/sub basics (`pubnub.publish(`, `pubnub.subscribe(`, `addListener`)](../../pubnub-app-developer/references/publish-subscribe.md). Real-time delivery follows [SDK initialization (`new PubNub(`, `userId`/UUID)](../../pubnub-app-developer/references/sdk-patterns.md). Reactions tied to a user honor [App Context user metadata](../../pubnub-app-context/references/users.md). For [Message Persistence and `fetchMessages`](../../pubnub-history/references/pagination-and-ordering.md) and [retention](../../pubnub-history/references/retention-and-storage.md) see `pubnub-history`. To route reaction events to external systems use [Events & Actions action targets](../../pubnub-events-and-actions/references/event-types.md).

# Message Actions

Message Actions let you attach metadata to a previously published message — typically reactions, edits, deletes, and read receipts. They are a separate stream from the original message and have their own listener.

## What a Message Action Is

A Message Action is a tuple of:
- `messageTimetoken` — the timetoken of the original message
- `type` — the category (e.g., `reaction`, `receipt`, `edited`, `deleted`)
- `value` — the value within that category (e.g., `:thumbsup:`, `read`, the new text)
- `actionTimetoken` — when the action was added
- `uuid` — who added it

## Add a Reaction

```javascript
await chat.sdk.addMessageAction({
  channel: channel.id,
  messageTimetoken: '17000000000000000',
  action: { type: 'reaction', value: ':thumbsup:' }
});
```

In the high-level Chat SDK:

```javascript
const message = /* a Message instance */;
await message.toggleReaction({ value: ':thumbsup:' });
```

## Listen for Actions

```javascript
pubnub.addListener({
  messageAction: (event) => {
    console.log(event.data); // { type, value, messageTimetoken, actionTimetoken, uuid }
  }
});
```

The Chat SDK exposes this via `channel.streamUpdates`.

## Edit a Message

By convention, edits use `type: 'edited'` with `value: '<new text>'`. The Chat SDK provides `message.editText(...)` which encodes this convention.

```javascript
await message.editText('updated text');
```

## Delete a Message

Two flavors:

1. **Soft delete** — add a `type: 'deleted'` action. UI hides the message; history still has it.
2. **Hard delete** — call `pubnub.deleteMessages({ channel, start, end })`. Removes from history. See the [history retention owner](../../pubnub-history/references/retention-and-storage.md).

```javascript
await message.delete();
```

## Read Receipts

Use `type: 'receipt'`, `value: 'read'`, or use `chat.markAllMessagesAsRead()`:

```javascript
await chat.markAllMessagesAsRead();
```

## Constraints

- Message Actions require **Message Persistence** to be enabled on the keyset (see [history retention](../../pubnub-history/references/retention-and-storage.md)).
- Actions are tied to a `messageTimetoken`; if the original message is hard-deleted, its actions go too.
- Multiple users can add the same `(type, value)` to the same message — the result is per-user, not coalesced.
- When fetching history with `fetchMessages`, pass `includeMessageActions: true` to get actions in the same call.

## Common Pitfalls

| Pitfall | Mitigation |
|---|---|
| Reaction looks like it didn't post | Confirm Message Persistence is on |
| Same user can react twice | Use `toggleReaction` from the Chat SDK, not raw `addMessageAction` |
| Read receipt not showing for offline users | Read receipts are stored as Message Actions; fetch on reconnect (see [offline catch-up](../../pubnub-history/references/offline-catch-up.md)) |
| Reaction storms in a busy channel | Coalesce client-side updates; don't render every event individually (see [payload hygiene](../../pubnub-observability/references/cost-and-payload-hygiene.md)) |
