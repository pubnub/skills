<!-- canonical-for: MESSAGE_FILTERS -->
<!-- used-by: pubnub-reliability -->

> **Cross-references:** Filter scoping for [Access Manager](../../pubnub-security/references/access-manager.md)-protected channels. Filtering reduces [transaction count / billing metrics](../../pubnub-observability/references/usage-metrics.md).

# Message Filters (`subscribeFilterExpression`)

The canonical reference for server-side filtering with PubNub's subscribe filter expression — narrowing what a subscriber receives without consuming the message client-side.

## What It Is

Subscribers can pass a `filterExpression` (or set `pubnub.setFilterExpression(...)`) so that PubNub evaluates a predicate **server-side** against each message's `meta` field. Messages that don't match are not delivered.

This is different from:

- **Client-side filtering**: receiving every message and dropping unwanted ones (wasteful — counts as a transaction).
- **Channel-level filtering**: putting different topics on different channels (better, but coarser).
- **[Events & Actions JSONPath filtering](../../pubnub-events-and-actions/references/filters-and-jsonpath.md)**: server-side, but for routing to external systems.

`filterExpression` operates on metadata only, not on the message payload.

## Setting the Filter

### At Initialization

```javascript
const pubnub = new PubNub({
  publishKey: process.env.PN_PUBLISH_KEY,
  subscribeKey: process.env.PN_SUBSCRIBE_KEY,
  userId: getUserId(),
  filterExpression: "region == 'us-west' && priority == 'high'"
});
```

### Dynamically

```javascript
pubnub.setFilterExpression("region == 'us-east'");
// Filter applies to all subsequent receives on this client
```

`setFilterExpression('')` clears the filter.

## Filter Expression Syntax

Operators on metadata fields:

```
==, !=
<, <=, >, >=
contains
LIKE
&&, ||, !
```

Examples:

```
region == 'us-west'
priority >= 5
type contains 'alert'
type LIKE 'sport-*'
region == 'us-west' && priority == 'high'
type != 'system' && (region == 'us-west' || region == 'us-east')
```

## Publisher Side: Setting `meta`

The filter sees only the `meta` parameter to `publish`, not the message payload itself. Producers must set `meta` with the fields the filter will reference:

```javascript
await pubnub.publish({
  channel: 'alerts',
  message: { text: 'High latency detected' },
  meta: { region: 'us-west', priority: 'high', type: 'alert' }
});
```

`meta` is published alongside the message and visible to filter expressions on every subscriber.

## Why Use Filters

| Reason | Benefit |
|---|---|
| Reduce client-side processing | Subscribers only see relevant messages |
| Reduce bandwidth | Server doesn't deliver filtered messages |
| Per-user personalization | Each subscriber sets their own filter |
| Reduce per-receive transaction count | Filtered messages don't count |

## When NOT to Use Filters

| Case | Better approach |
|---|---|
| Filter shape changes per published message | Use payload-based logic in your handler |
| Filter must match on payload contents | Use a [Function](../../pubnub-functions/references/functions-basics.md) to inspect/transform |
| Filter must scale to thousands of distinct rules | Switch to channel-level routing or channel groups ([scaling-patterns.md](../../pubnub-scale/references/scaling-patterns.md)) |
| Need to filter for downstream system delivery | Use [Events & Actions JSONPath](../../pubnub-events-and-actions/references/filters-and-jsonpath.md) |

## Filter vs Channel Choice

A common design question: should this be a separate channel or a filtered channel?

| Use a separate channel when | Use a filter when |
|---|---|
| The filter criterion is fixed and known up front (e.g., `room-42`) | Each subscriber wants a personalized slice of a shared stream |
| Cardinality is low (5–50 channels) | Cardinality would be very high (thousands of "channels") |
| You need different ACLs / Access Manager grants | Same ACL applies to all messages |
| Producers know the audience at publish time | Producers don't know who will subscribe |

## Common Anti-Patterns

| Anti-pattern | Fix |
|---|---|
| Filtering on payload (`message.region`) instead of `meta.region` | Move the field to `meta` |
| Heavy client-side filter when a server filter would work | Push it server-side |
| Forgetting to update `setFilterExpression` after user changes preferences | Re-set on every preference change |
| Filter expression syntax errors (silent — server matches nothing) | Test with a known-good message |
| Putting too many AND/OR clauses (deeply nested) | Simplify; complex filters slow eval |

## Related Reading

- [publish-subscribe.md](publish-subscribe.md) — basic subscribe semantics
- [channels.md](channels.md) — channel-vs-filter design choice
- [pubnub-events-and-actions/references/filters-and-jsonpath.md](../../pubnub-events-and-actions/references/filters-and-jsonpath.md) — filtering for external delivery
