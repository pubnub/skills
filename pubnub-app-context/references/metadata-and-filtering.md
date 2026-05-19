<!-- canonical-for: APP_CONTEXT_FILTERING -->
<!-- used-by: -->

# App Context: Filtering, Sorting, Pagination, and Change Events

The canonical reference for efficient querying of App Context objects and reacting to metadata changes in real time.

## Filter Expressions

App Context filters are evaluated server-side, returning only matching records. Always prefer a filter over fetching everything and filtering client-side.

### Operators

| Operator | Meaning | Example |
|---|---|---|
| `==` | Equality | `name == "Alice"` |
| `!=` | Inequality | `custom.role != "guest"` |
| `<`, `<=`, `>`, `>=` | Numeric / lexical comparison | `custom.reputation > 100` |
| `LIKE` | Wildcard match (`*` for any chars) | `name LIKE "Al*"` |
| `&&` | Logical AND | `name LIKE "A*" && custom.verified == true` |
| `\|\|` | Logical OR | `custom.role == "admin" \|\| custom.role == "moderator"` |
| `!` | Logical NOT | `!(custom.muted == true)` |

### Field Path Syntax

- Top-level fields: `name`, `email`, `description`, `updated`
- Custom fields: `custom.<field_name>`
- Embedded objects (in memberships): `channel.name`, `channel.custom.<field>`, `uuid.name`, `uuid.custom.<field>`

### Examples

```javascript
// All admins
filter: 'custom.role == "admin"'

// All channels updated in the last hour
filter: 'updated >= "2026-04-29T15:32:00Z"'

// Memberships where the channel is a photo challenge AND the user is active
filter: 'channel.custom.challenge_type == "photo" && custom.status == "active"'

// Users with a verified profile and high reputation
filter: 'custom.verified == true && custom.reputation > 1000'

// Channels whose name starts with "room-"
filter: 'name LIKE "room-*"'
```

## Sorting

```javascript
sort: { name: 'asc' }                       // single sort by name ascending
sort: { updated: 'desc', name: 'asc' }      // multi-key sort
sort: { 'custom.reputation': 'desc' }       // sort by a custom field
```

Allowed sort fields depend on the object type — `id`, `name`, `updated` are universally available; `custom.*` is supported but may be slower at scale.

## Pagination

App Context uses cursor-based pagination via `next` and `prev` tokens:

```javascript
let cursor;
do {
  const result = await pubnub.objects.getAllUUIDMetadata({
    include: { customFields: true, totalCount: true },
    filter: 'custom.role == "admin"',
    sort: { name: 'asc' },
    limit: 100,
    page: cursor ? { next: cursor } : undefined
  });

  result.data.forEach(processUser);
  cursor = result.next;
} while (cursor);
```

`limit` defaults to 100 and is capped at 100 per request. Always pass an explicit limit so future SDK changes don't surprise you.

## Bounded Queries — Avoid `getAll` Without a Filter

`getAllUUIDMetadata`, `getAllChannelMetadata` without a filter return *every* record. At scale this is slow and expensive.

Apply a filter for any production read:

```javascript
// Bad
const all = await pubnub.objects.getAllUUIDMetadata({ limit: 100 });

// Good
const recent = await pubnub.objects.getAllUUIDMetadata({
  filter: 'updated >= "2026-04-29T00:00:00Z"',
  limit: 100
});
```

## Metadata Change Events

When App Context Metadata Events are enabled in the Admin Portal, every change publishes an event on a system channel. Subscribe to react in real time.

### Event Types

| `type` | `event` values | Fired when |
|---|---|---|
| `uuid` | `set`, `delete` | A user object is upserted or removed |
| `channel` | `set`, `delete` | A channel object is upserted or removed |
| `membership` | `set`, `delete` | A membership is upserted or removed |

### Listening

```javascript
pubnub.addListener({
  objects: (event) => {
    const { type, event: action, data } = event.message;

    if (type === 'uuid' && action === 'set') {
      cacheUser(data);
    } else if (type === 'channel' && action === 'delete') {
      removeChannelFromUI(data.id);
    } else if (type === 'membership' && action === 'set') {
      updateRoomMembershipBadge(data);
    }
  }
});

pubnub.subscribe({ channels: ['user-123', 'room-photo-challenge-123'] });
```

Events are published on the corresponding object's channel — to receive user-123's metadata events, subscribe to channel `user-123`. See [userId conventions](../../pubnub-app-developer/references/sdk-patterns.md) and [channel name rules](../../pubnub-app-developer/references/channels.md) for the underlying primitives.

For [`addListener` mechanics](../../pubnub-app-developer/references/publish-subscribe.md) see the canonical owner.

## Bulk Operations

Memberships support bulk add/remove in a single call:

```javascript
// Bulk add user to many channels
await pubnub.objects.setMemberships({
  uuid: 'user-123',
  channels: [
    { id: 'room-1', custom: { role: 'member' } },
    { id: 'room-2', custom: { role: 'member' } },
    { id: 'room-3', custom: { role: 'admin' } }
  ]
});

// Bulk remove
await pubnub.objects.removeMemberships({
  uuid: 'user-123',
  channels: ['room-1', 'room-2']
});
```

User and channel objects do not support bulk set/delete — each requires its own call.

## Pagination + Filtering Cookbook: Server-Side User Search

```javascript
async function* searchUsers(prefix) {
  let cursor;
  do {
    const result = await pubnub.objects.getAllUUIDMetadata({
      include: { customFields: true },
      filter: `name LIKE "${prefix}*"`,
      sort: { name: 'asc' },
      limit: 100,
      page: cursor ? { next: cursor } : undefined
    });
    for (const user of result.data) yield user;
    cursor = result.next;
  } while (cursor);
}

// Usage
for await (const user of searchUsers('Al')) {
  console.log(user.name, user.custom?.role);
}
```

## Common Anti-Patterns

| Anti-pattern | Fix |
|---|---|
| `getAll*` without a filter at scale | Always filter; even `updated >= timestamp` is fine |
| Client-side filtering of large lists | Push the predicate into `filter:` |
| Not setting `limit` explicitly | Always pass an explicit `limit` (max 100) |
| Reading without `include.customFields: true` and being confused that `custom` is missing | Always include when custom matters |
| Subscribing to a "metadata events" channel that doesn't exist | Subscribe to the object's own channel (the userId or channelId) |

## Related Reading

- [users.md](users.md) — User Objects
- [channels-and-memberships.md](channels-and-memberships.md) — Channel and Membership objects
