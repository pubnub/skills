<!-- canonical-for: APP_CONTEXT -->
<!-- used-by: pubnub-choose-docs-path, pubnub-keyset-management, pubnub-chat, pubnub-illuminate -->

# App Context: Users (UUID Metadata)

The canonical reference for storing, querying, and managing user profile data in PubNub App Context.

## What a User Object Is

A User Object stores metadata about an individual identified by [`userId` (also known as UUID)](../../pubnub-app-developer/references/sdk-patterns.md). The same `userId` value identifies the user across all PubNub APIs (publish, subscribe, presence, App Context, [Access Manager](../../pubnub-security/references/access-manager.md)).

Standard fields:

| Field | Type | Purpose |
|---|---|---|
| `id` | string | The userId. Required, unique, persistent. |
| `name` | string | Display name |
| `email` | string | Email |
| `externalId` | string | Reference to your own user record (foreign key) |
| `profileUrl` | string | Avatar / profile picture URL |
| `custom` | object | Application-specific scalar fields |
| `eTag` | string | Server-managed concurrency token |
| `updated` | timestamp | Server-managed last-update time |

## Create / Update a User

Set is upsert — same call creates or updates.

```javascript
await pubnub.objects.setUUIDMetadata({
  uuid: 'user-123',
  data: {
    name: 'Alice Liddell',
    email: 'alice@example.com',
    externalId: 'crm-987654',
    profileUrl: 'https://cdn.example.com/avatars/123.jpg',
    custom: {
      role: 'admin',
      timezone: 'America/Los_Angeles',
      theme_preference: 'dark',
      notifications_enabled: true,
      app_version: '2.1.0'
    }
  }
});
```

If `uuid` is omitted, the SDK uses the userId from PubNub initialization (which is the most common pattern — a user manages their own metadata).

## Read a User

Always pass `include: { customFields: true }` if you intend to read `custom`. Without it, the response omits `custom`.

```javascript
const result = await pubnub.objects.getUUIDMetadata({
  uuid: 'user-123',
  include: { customFields: true }
});

console.log(result.data.name);            // "Alice Liddell"
console.log(result.data.custom.role);     // "admin"
console.log(result.data.eTag);            // server token; pass back on next set for optimistic concurrency
```

## List All Users

```javascript
const result = await pubnub.objects.getAllUUIDMetadata({
  include: { customFields: true, totalCount: true },
  filter: 'custom.role == "admin"',
  sort: { name: 'asc' },
  limit: 100
});

console.log(`Total admins: ${result.totalCount}`);
result.data.forEach(user => console.log(user.id, user.name, user.custom.role));
```

For filter expression syntax see [metadata-and-filtering.md](metadata-and-filtering.md).

## Delete a User

```javascript
await pubnub.objects.removeUUIDMetadata({ uuid: 'user-123' });
```

If Referential Integrity is enabled in the Admin Portal, deleting a user also removes all their memberships. If it is off, dangling memberships will exist and need to be cleaned up separately.

## Custom Field Schema Patterns

### Use Scalar Values

```javascript
custom: {
  role: 'moderator',          // string
  reputation: 1250,           // number
  verified: true,             // boolean
  joined_at: '2026-04-29T16:32:00Z'  // ISO timestamp string
}
```

### Shallow Nested Objects Allowed

```javascript
custom: {
  preferences: {
    notifications: true,
    theme: 'dark',
    language: 'en'
  }
}
```

### Avoid Deep Nesting

App Context is a directory, not a JSON document store. Don't store deep objects, arrays of records, or anything that wants to be a relation. Move that into your own database and store only the foreign key (`externalId`) here.

### Versioning the Custom Schema

Add an `app_version` (or `schema_version`) field to every record so you know which client wrote it. This makes migrations far easier:

```javascript
custom: {
  schema_version: 3,
  // ... your fields
}
```

## Concurrency: Use eTag

When two clients update the same user simultaneously, the second write can clobber the first. Pass the `eTag` from your read into your set to opt into optimistic concurrency:

```javascript
const current = await pubnub.objects.getUUIDMetadata({ uuid: 'user-123', include: { customFields: true } });

await pubnub.objects.setUUIDMetadata({
  uuid: 'user-123',
  data: {
    name: 'Alice L.',
    custom: { ...current.data.custom, theme_preference: 'light' }
  },
  ifMatchesEtag: current.data.eTag
});
```

If the eTag doesn't match (another writer beat you), the set fails — handle by re-reading and retrying.

## Listening for User Metadata Changes

Subscribe to user metadata events on a listener:

```javascript
pubnub.addListener({
  objects: (event) => {
    if (event.message.type === 'uuid' && event.message.event === 'set') {
      console.log('User updated:', event.message.data.id);
    }
    if (event.message.type === 'uuid' && event.message.event === 'delete') {
      console.log('User deleted:', event.message.data.id);
    }
  }
});

pubnub.subscribe({ channels: ['user-123'] });  // events for this user are fanned out on the user channel
```

For the basic [`addListener` and `subscribe` mechanics](../../pubnub-app-developer/references/publish-subscribe.md) see the canonical owner.

## Common Anti-Patterns

| Anti-pattern | Fix |
|---|---|
| Forgetting `include.customFields: true` on read → `custom` is undefined | Always include on reads that touch `custom` |
| Storing large arrays or deep JSON in `custom` | Move to your own DB; keep a foreign key reference |
| Using App Context as the primary user store | App Context is a directory — your DB is the source of truth |
| Updating without eTag → silent overwrites | Use `ifMatchesEtag` for any user-editable field |
| Forgetting to clean up memberships when Referential Integrity is off | Enable Referential Integrity in the Admin Portal |
| Hardcoding `userId` in tests | Use a unique test prefix per run to avoid pollution |

## Related Reading

- [channels-and-memberships.md](channels-and-memberships.md) — channel objects and user-channel relationships
- [metadata-and-filtering.md](metadata-and-filtering.md) — filter syntax, sorting, pagination, change events
- [pubnub-keyset-management/references/keysets-and-environments.md](../../pubnub-keyset-management/references/keysets-and-environments.md) — enabling the App Context add-on
