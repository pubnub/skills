# PubNub Chat SDK Patterns

## User lifecycle

| Pattern | Rule |
|---------|------|
| Get-or-create | Resolve user by ID before channel creation; create only when missing |
| Profile updates | Current user vs other-user updates depend on App Context permissions |
| List/filter users | Use App Context filters for online rosters; defer filter syntax to Chat SDK docs |

## Channel lifecycle

| Pattern | Rule |
|---------|------|
| Direct channel cache | Cache by sorted member pair key; search existing DMs before `createDirectConversation` |
| Membership listing | Drive inbox from `currentUser.getMemberships`, not ad-hoc channel IDs |
| Join/leave | Public channels: explicit join; private: grant + invite flow |

## Real-time UI orchestration

Typical React (or equivalent) sequence:

1. Load channel + history chunk on mount
2. `channel.connect` for live messages; unsubscribe on unmount
3. Typing: `startTyping` / `stopTyping` with debounce; filter self from `onTyping`
4. Send: `sendText` then clear input and stop typing
5. Logout: `chat.disconnect`, clear caches, tear down listeners

Pair live delivery with [offline catch-up](../../pubnub-history/references/offline-catch-up.md) and [dedup-on-merge](../../pubnub-reliability/references/dedup-on-merge.md) for reconnect gaps.

## Notifications

| Pattern | Rule |
|---------|------|
| Unread badges | Derive from membership unread counts; refresh on message events, not constant polling |
| Mark read | When channel becomes visible, set last-read to latest message via membership API |
| Global read | `markAllMessagesAsRead` for inbox-zero flows |

## Threading UI

Load thread via `parentMessage.getThread()` when `hasThread`; subscribe on thread channel for replies. Lazy-create with `createThread(text)` on first reply. See [threading.md](threading.md).

## Performance

1. **Cache channels** — avoid recreating channel objects each render
2. **Paginate history** — load in chunks, not full backlog
3. **Debounce typing** — do not signal on every keystroke
4. **Virtualize long lists** — window message rendering
5. **Batch state updates** — reduce re-renders on burst traffic
6. **Cleanup subscriptions** — always disconnect on unmount
