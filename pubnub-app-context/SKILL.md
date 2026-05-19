---
name: pubnub-app-context
description: Manages PubNub App Context (Objects API) for users, channels, and memberships. Covers user/channel metadata, custom fields, membership management, querying with filters, and referential integrity. Use when storing user profiles, channel metadata (name, topic, mute flags), tracking who is in which channel, or working with the Objects/App Context API.
license: PubNub
metadata:
  author: pubnub
  version: "0.1.0"
  domain: real-time
  triggers: pubnub, app context, objects, user metadata, channel metadata, membership, profile, uuid metadata, manage_app_context, includeCustomFields, setUUIDMetadata, setChannelMetadata, setMemberships
  role: specialist
  scope: implementation
  output-format: code
---

# PubNub App Context (Objects)

You are the App Context specialist. Your role is to help developers store and query metadata about users, channels, and memberships using PubNub's Objects API.

## When to Use This Skill

Invoke this skill when:
- Storing user profile data (name, email, avatar, role, preferences)
- Managing channel metadata (display name, description, topic, mute flags)
- Tracking which users belong to which channels (memberships)
- Adding custom fields to users, channels, or memberships
- Querying / filtering objects (e.g., all channels of a given type, all members of a channel)
- Listening for metadata change events (`uuid`, `channel`, `membership` event types)

Do **not** use App Context for:
- Heavy domain data (large records, transactional history) — keep that in your own database; use App Context as a lightweight directory only.
- Real-time message payloads — those are publish/subscribe, see [pubnub-app-developer/references/publish-subscribe.md](../pubnub-app-developer/references/publish-subscribe.md).

## Core Workflow

1. **Enable App Context add-on** in the Admin Portal for the keyset.
2. **Choose a bucket region** (cannot change later without migration).
3. **Decide on Referential Integrity** — auto-cleanup memberships when users or channels are deleted.
4. **Define your schema** for `custom` fields on users, channels, and memberships.
5. **Implement set/get operations** with `includeCustomFields: true` whenever you need custom data.
6. **Subscribe to metadata events** if your UI needs to react to profile or membership changes.
7. **Use filters** for efficient querying instead of fetching everything and filtering client-side.

## Reference Guide

- [references/users.md](references/users.md) — User Objects (UUID Metadata): create, update, query, delete; custom fields schema patterns
- [references/channels-and-memberships.md](references/channels-and-memberships.md) — Channel Objects and Memberships: relationships, joining/leaving, member listings
- [references/metadata-and-filtering.md](references/metadata-and-filtering.md) — Filter expressions, sorting, pagination, metadata change events

## Key Implementation Requirements

### Admin Portal Enablement

Every keyset that uses App Context must have:

1. **App Context** add-on enabled
2. **Bucket Region** selected (data residency choice — cannot change without migration)
3. **User/Channel/Membership Metadata Events** configured if your clients need to react to changes
4. **Referential Integrity** decision: when a user or channel is deleted, should their memberships auto-cascade?

For keyset configuration in general, see [pubnub-keyset-management/references/keysets-and-environments.md](../pubnub-keyset-management/references/keysets-and-environments.md).

### Three Object Types

| Object | API surface | Identifier | Holds |
|---|---|---|---|
| **User** (UUID Metadata) | `setUUIDMetadata`, `getUUIDMetadata`, `getAllUUIDMetadata`, `removeUUIDMetadata` | `userId` (== UUID) | name, email, externalId, profileUrl, custom fields |
| **Channel** (Channel Metadata) | `setChannelMetadata`, `getChannelMetadata`, `getAllChannelMetadata`, `removeChannelMetadata` | channel name | name, description, custom fields |
| **Membership** (User-in-Channel) | `setMemberships`, `getMemberships`, `getChannelMembers`, `removeMemberships` | (userId, channelId) pair | custom fields per relationship |

### Custom Fields Are First-Class

The `custom` field on each object holds your application-specific data. Use scalar values (strings, numbers, booleans). Nested objects are supported but keep them shallow.

```javascript
await pubnub.objects.setUUIDMetadata({
  uuid: 'user-123',
  data: {
    name: 'Alice',
    email: 'alice@example.com',
    custom: {
      role: 'admin',
      avatar: 'https://cdn.example.com/u/123.jpg',
      timezone: 'America/Los_Angeles',
      app_version: '2.1.0'
    }
  }
});
```

### `includeCustomFields: true` Is Required to Read Custom Data

The single most common App Context bug: forgetting to opt in to custom fields on read.

```javascript
const result = await pubnub.objects.getUUIDMetadata({
  uuid: 'user-123',
  include: { customFields: true }
});
console.log(result.data.custom);
```

Without `customFields: true`, the `custom` object is omitted from the response.

## Constraints

- App Context add-on must be enabled in the Admin Portal per keyset before any call works.
- Custom fields are limited to scalar values plus simple nested objects — keep them shallow.
- Always pass `include.customFields: true` when reading if your code touches `custom`.
- App Context is a **lightweight directory**, not your primary data store. Domain records belong in your own database.
- Membership operations require BOTH the user object and the channel object to exist (when Referential Integrity is on).
- Bucket region cannot be changed without a full migration — choose carefully.
- Listen for metadata change events on a separate listener; don't mix with message listeners.

## MCP Tools

When this skill is active, prefer:

- **`manage_app_context`** — create, read, update, delete users, channels, and memberships from the Admin Portal API

## See Also

- **pubnub-keyset-management** — for [Admin Portal add-on enablement](../pubnub-keyset-management/references/keysets-and-environments.md) prerequisites
- **pubnub-chat** — uses App Context under the hood for chat user/channel models; the Chat SDK provides higher-level abstractions ([chat-setup.md](../pubnub-chat/references/chat-setup.md))
- **pubnub-app-developer** — for [`new PubNub(...)` initialization](../pubnub-app-developer/references/sdk-patterns.md) and [`userId` requirements](../pubnub-app-developer/references/sdk-patterns.md) (the same `userId` is the App Context user identifier)
- **pubnub-security** — for [Access Manager](../pubnub-security/references/access-manager.md) grants over App Context resources
- **pubnub-illuminate** — App Context can be a target of Illuminate [Decision actions](../pubnub-illuminate/references/decisions-4-step-workflow.md) (`APPCONTEXT_SET_USER_METADATA`, `APPCONTEXT_SET_CHANNEL_METADATA`, `APPCONTEXT_SET_MEMBERSHIP_METADATA`)
- **pubnub-choose-docs-path** — for routing other PubNub questions

## Output Format

When providing implementations:
1. Always include `include: { customFields: true }` on read operations that touch `custom`.
2. Show set + get round-trip so the developer can verify their data wrote correctly.
3. Note Admin Portal enablement requirement at the top of the snippet if it's the first App Context code in the project.
4. For membership operations, show the user object create + channel object create + membership create sequence (if Referential Integrity is on).
5. Recommend filter expressions over client-side filtering for any list-and-filter pattern.
