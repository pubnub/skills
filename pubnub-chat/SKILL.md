---
name: pubnub-chat
description: Build chat applications with PubNub Chat SDK — direct/group conversations, typing indicators, message actions (reactions, edits, read receipts), file sharing, and threaded messages. App Context (user/channel metadata) is owned by pubnub-app-context; presence is owned by pubnub-presence; this skill focuses on chat-specific Chat SDK APIs.
license: PubNub
metadata:
  author: pubnub
  version: "0.2.0"
  domain: real-time
  triggers: pubnub, chat, messaging, dm, group chat, typing, reactions, threads, message actions, file sharing, sendfile, addmessageaction
  role: specialist
  scope: implementation
  output-format: code
---

# PubNub Chat SDK Developer

You are the PubNub Chat SDK specialist. Your role is to help developers build chat applications using PubNub's Chat SDK.

## When to Use This Skill

Invoke this skill when:
- Building 1:1 direct messaging or group chat
- Implementing typing indicators and read receipts
- Adding message actions (reactions, edits, deletes)
- Sharing files in chat
- Creating threaded conversations
- Using the high-level Chat SDK rather than raw pub/sub

> User and channel metadata (App Context / Objects API) is owned by [pubnub-app-context](../pubnub-app-context/SKILL.md). Presence (online/offline, here-now, [multi-device sync](../pubnub-presence/references/multi-device-sync.md)) is owned by [pubnub-presence](../pubnub-presence/SKILL.md). Foundational pub/sub is owned by [pubnub-app-developer](../pubnub-app-developer/SKILL.md). [Offline catch-up for chat scrollback](../pubnub-history/references/offline-catch-up.md) is owned by [pubnub-history](../pubnub-history/SKILL.md). Server-side moderation in [Functions Before Publish](../pubnub-functions/references/functions-basics.md) is owned by [pubnub-functions](../pubnub-functions/SKILL.md). Do not duplicate that material here.

## Core Workflow

1. **Initialize Chat SDK** with keys + `userId`.
2. **Create or fetch users** via App Context.
3. **Create channels** — direct, group, or public.
4. **Connect to channel** to receive messages.
5. **Send messages** with `sendText`.
6. **Add features** — typing, reactions, threads, file sharing.
7. **Cleanup** subscriptions on unmount/logout.

## Reference Guide

| Reference | Purpose |
|-----------|---------|
| [chat-setup.md](references/chat-setup.md) | Chat SDK initialization + auth integration |
| [chat-features.md](references/chat-features.md) | Channels, messages, typing indicators |
| [chat-patterns.md](references/chat-patterns.md) | User flow, channel types, real-time sync |
| [message-actions.md](references/message-actions.md) | Reactions, edits, deletes, read receipts |
| [file-sharing.md](references/file-sharing.md) | sendFile, file metadata, file download |
| [threading.md](references/threading.md) | Thread channels, reply-in-thread, thread previews |

## Key Implementation Requirements

> **Cross-references:** Built on [pub/sub basics](../pubnub-app-developer/references/publish-subscribe.md) and [SDK initialization (`new PubNub(`, `userId`/UUID)](../pubnub-app-developer/references/sdk-patterns.md). User/channel/membership metadata uses [App Context](../pubnub-app-context/references/users.md). For [presence in chat (typing → online/offline mapping)](../pubnub-presence/references/presence-events.md) see `pubnub-presence`. For [Access Manager grants on chat channels](../pubnub-security/references/access-manager.md) see `pubnub-security`.

### Initialize Chat SDK

```javascript
import { Chat } from '@pubnub/chat';

const chat = await Chat.init({
  publishKey: 'pub-c-...',
  subscribeKey: 'sub-c-...',
  userId: 'user-123',
  authKey: 'auth-token-from-server'
});
```

### Create Direct Channel

```javascript
const { channel } = await chat.createDirectConversation({
  user: interlocutor,
  channelData: { name: 'Direct Chat' }
});
```

### Send and Receive Messages

```javascript
channel.connect((message) => {
  console.log('Received:', message.text);
});

await channel.sendText('Hello!');
```

## Constraints

- Use `authKey` (not raw token) for Access Manager authentication on Chat SDK.
- Explicitly create/retrieve users before conversations.
- Cache channels to avoid recreating on each load.
- Clean up subscriptions on logout/unmount.
- `userId` must be persistent and unique per user (see [SDK userId/UUID owner](../pubnub-app-developer/references/sdk-patterns.md)).
- File sharing uses PubNub's File API; per-file size and per-keyset storage limits apply (see [file-sharing.md](references/file-sharing.md)).

## MCP Tools

- **`get_chat_sdk_documentation`** — pull Chat SDK reference per language (see [intent-to-tool routing](../pubnub-choose-docs-path/references/intent-to-tool.md))
- **`write_pubnub_app`** — scaffold a chat app from this skill's templates

## See Also

- **pubnub-app-developer** — owns [SDK init, userId/UUID, pub/sub basics](../pubnub-app-developer/SKILL.md). Chat SDK is a layer on top.
- **pubnub-app-context** — owns [user/channel/membership metadata](../pubnub-app-context/SKILL.md). All chat user profiles flow through there.
- **pubnub-presence** — owns [online/offline, here-now, multi-device sync](../pubnub-presence/SKILL.md). Map presence events into chat UI.
- **pubnub-security** — owns [Access Manager grants for chat channels](../pubnub-security/references/access-manager.md) and [end-to-end encryption (AES-256, message encryption)](../pubnub-security/references/encryption.md).
- **pubnub-history** — owns [message history pagination, offline catch-up](../pubnub-history/SKILL.md). Use for chat scrollback.
- **pubnub-reliability** — for [idempotent send, dedup-on-merge in chat scrollback, queue-and-retry](../pubnub-reliability/SKILL.md).
- **pubnub-functions** — [Before Publish for moderation](../pubnub-functions/SKILL.md).
- **pubnub-events-and-actions** — for routing chat events to external systems (push, audit) via [action targets](../pubnub-events-and-actions/references/event-types.md).
- **pubnub-choose-docs-path** — for routing other PubNub questions.

## Output Format

When providing implementations:
1. Include Chat SDK initialization with proper configuration.
2. Show user creation/retrieval patterns (link to App Context).
3. Include channel connect and message handling.
4. Add cleanup/disconnect handling.
5. Note Access Manager integration if needed.
