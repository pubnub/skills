# Message Actions

Message Actions attach metadata to a previously published message — reactions, edits, deletes, read receipts. They are a separate stream from the original message and have their own listener.

## What a Message Action Is

A Message Action is keyed by:

- `messageTimetoken` — the original message
- `type` — category (e.g., `reaction`, `receipt`, `edited`, `deleted`)
- `value` — value within that category
- `actionTimetoken`, `uuid` — when and who

## Capability selection

| Need | Path |
|------|------|
| Reactions in Chat SDK | Prefer `message.toggleReaction()` |
| Raw Core SDK actions | `pubnub.addMessageAction` — retrieve schema via **`get_sdk_documentation`** |
| Live updates | Core: `messageAction` listener; Chat: `channel.streamUpdates` |
| Edit | Chat: `message.editText()`; convention is `type: 'edited'` |
| Soft delete | `type: 'deleted'` action — UI hides; history retains |
| Hard delete | `deleteMessages` — see [retention owner](../../pubnub-history/references/retention-and-storage.md) |
| Read receipts | Requires Unread Message Count setup; mark via membership `setLastReadMessage` / `markAllMessagesAsRead`; listen with `onReadReceiptReceived` |

## Constraints

- Message Actions require **Message Persistence** on the keyset.
- Actions bind to `messageTimetoken`; hard-deleting the message removes its actions.
- Multiple users can add the same `(type, value)` — per-user, not coalesced.
- When fetching history, request actions in the same call (`includeMessageActions: true`).

## Common Pitfalls

| Pitfall | Mitigation |
|---|---|
| Reaction looks like it didn't post | Confirm Message Persistence is on |
| Same user can react twice | Use Chat SDK `toggleReaction`, not raw duplicate `addMessageAction` |
| Read receipt not showing offline | Actions are persisted; fetch on reconnect ([offline catch-up](../../pubnub-history/references/offline-catch-up.md)) |
| Reaction storms in busy channel | Coalesce client-side ([payload hygiene](../../pubnub-observability/references/cost-and-payload-hygiene.md)) |
