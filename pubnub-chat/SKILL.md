---
name: pubnub-chat
description: Build chat applications with PubNub Chat SDK — direct/group conversations, typing indicators, message actions (reactions, edits, read receipts), file sharing, and threaded messages. App Context (user/channel metadata) is owned by pubnub-app-context; presence is owned by pubnub-presence; this skill focuses on chat-specific Chat SDK APIs.
license: PubNub
metadata:
  author: pubnub
  version: "0.3.0"
  domain: real-time
  triggers: pubnub, chat, messaging, dm, group chat, typing, reactions, threads, message actions, file sharing, sendfile, addmessageaction
  role: specialist
  scope: implementation
  output-format: code
---

# PubNub Chat SDK Developer

You are the PubNub Chat SDK specialist. Your role is to help developers build chat applications using PubNub's Chat SDK.

> **Precedence:** PubNub MCP tools and pubnub.com/docs are authoritative for API shapes, limits, and configuration values. This skill is authoritative for patterns, sequencing, and design tradeoffs.



## When to Use This Skill

Invoke this skill when:
- Building 1:1 direct messaging or group chat with the **Chat SDK** (not raw pub/sub)
- Implementing typing, reactions, threads, files, read receipts
- Orchestrating Chat SDK with App Context, AM, and history

> User/channel metadata → [pubnub-app-context](../pubnub-app-context/SKILL.md). Presence → [pubnub-presence](../pubnub-presence/SKILL.md). Raw pub/sub → [pubnub-app-developer](../pubnub-app-developer/SKILL.md). Offline scrollback → [pubnub-history](../pubnub-history/SKILL.md).

## Core Workflow

1. **Prerequisites** — App Context + Message Persistence enabled on keyset; retrieve init options via **`get_chat_sdk_documentation`**.
2. **Initialize** — `Chat.init` with keys + persistent `userId`; AM token via **`authKey`** (not `token`).
3. **Users** — create/fetch users before conversations (App Context under the hood).
4. **Channels** — direct (`createDirectConversation`), group (`createGroupConversation`), public (`createPublicConversation`) — retrieve API from Chat SDK docs.
5. **Connect & send** — `channel.connect` + `sendText`; cleanup on logout.
6. **Features** — typing, reactions, threads, files — use feature-specific Chat SDK docs via MCP.
7. **Validate** — round-trip message + history + AM grants on staging.

## Reference Guide

| Reference | Purpose |
|-----------|---------|
| [chat-patterns.md](references/chat-patterns.md) | Caching, React provider flow, conversation assembly |
| [message-actions.md](references/message-actions.md) | Reactions vs raw Message Actions; read receipt orchestration |
| [file-sharing.md](references/file-sharing.md) | File upload tradeoffs and limits |
| [threading.md](references/threading.md) | Thread lifecycle, billing, cleanup |

Retrieve channel/message API cookbooks from **`get_chat_sdk_documentation`** — not duplicated here.

## Key decisions

| Decision | Guidance |
|----------|----------|
| Chat SDK vs raw pub/sub | Use Chat SDK for DM/group/features; raw pub/sub for custom protocols |
| `authKey` vs `token` | Chat SDK expects **`authKey`** for AM — verify in Chat SDK config docs |
| User bootstrap | Always `getUser` → `createUser` if missing before conversations |
| Channel cache | Cache active conversations in memory to avoid duplicate creates |
| Public vs group | Public = open join; group = explicit membership |
| Threads | Each thread is a channel — lazy-create on first reply; call `removeThread` when cleaning up |

## Constraints

- Persistent `userId` ([pubnub-app-developer/SKILL.md](../pubnub-app-developer/SKILL.md)).
- Clean up `channel.connect` / `chat.disconnect` on unmount.
- File size limits — retrieve from Admin Portal / docs ([file-sharing.md](references/file-sharing.md)).
- Read receipts require unread-count setup first — see Chat SDK unread docs via MCP.

## MCP Tools

- **`get_chat_sdk_documentation`** — all Chat SDK method surfaces per language/feature
- **`write_pubnub_app`** — scaffold chat apps

## See Also

- **pubnub-app-context** — users, channels, memberships
- **pubnub-security** — AM grants per chat channel
- **pubnub-reliability** — dedup-on-merge for scrollback
- **pubnub-functions** — Before Publish moderation
- **pubnub-events-and-actions** — external routing of chat events

## Output Format

1. Retrieve Chat SDK APIs from MCP for the requested feature.
2. Show orchestration (init → users → channel → connect → feature).
3. Include cleanup and AM integration notes when relevant.
