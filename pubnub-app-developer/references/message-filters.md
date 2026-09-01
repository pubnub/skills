<!-- canonical-for: MESSAGE_FILTERS -->
<!-- used-by: pubnub-reliability -->

> **Cross-references:** Filter scoping for [Access Manager](../../pubnub-security/references/access-manager.md)-protected channels. Filtering reduces [transaction count / billing metrics](../../pubnub-observability/references/usage-metrics.md).

# Message Filters — Decisions and Tradeoffs


## What filters do

`subscribeFilterExpression` evaluates **server-side against message `meta` only** (not the payload). Non-matching messages are not delivered — they also **do not count** as receive transactions.

This differs from:

- **Client-side filtering** — wasteful; every message counts.
- **Separate channels** — coarse but simple ACL and routing.
- **Events & Actions JSONPath** — downstream delivery filtering ([filters-and-jsonpath.md](../../pubnub-events-and-actions/references/filters-and-jsonpath.md)).
- **Functions Before Publish** — inspect/transform payload content.

## Why use filters

| Reason | Benefit |
|--------|---------|
| Reduce client processing | Subscribers only see relevant messages |
| Reduce bandwidth | Server drops non-matches |
| Per-user personalization | Each client sets its own expression |
| Billing | Filtered messages don't count as receives |

## When NOT to use filters

| Case | Better approach |
|------|-----------------|
| Criterion changes per published message shape | Handler logic or separate channels |
| Must match on payload contents | [Function](../../pubnub-functions/references/functions-basics.md) or channel routing |
| Thousands of distinct rules | Channel groups or hierarchical channels ([scaling-patterns.md](../../pubnub-scale/references/scaling-patterns.md)) |
| Filter for external systems | [Events & Actions JSONPath](../../pubnub-events-and-actions/references/filters-and-jsonpath.md) |

## Filter vs separate channel

| Use a separate channel when | Use a filter when |
|----------------------------|-------------------|
| Criterion is fixed up front (e.g. `room-42`) | Each subscriber wants a personalized slice |
| Low cardinality (5–50 channels) | High cardinality would explode channel count |
| Different ACLs per audience | Same ACL for all messages on the stream |
| Producers know subscribers at publish time | Producers broadcast; subscribers self-select |

## Anti-patterns

| Anti-pattern | Fix |
|--------------|-----|
| Filtering on payload fields | Move discriminant to `meta` at publish time |
| Heavy client-side drop | Push filter server-side |
| Stale expression after preference change | Re-`setFilterExpression` on every preference update |
| Syntax errors (silent no-match) | Test expression against known-good `meta` samples |
| Deeply nested expressions | Simplify; prefer channel split |

## Orchestration

1. Decide channel vs filter vs Function vs E&A for the routing need.
2. Retrieve current filter syntax from SDK docs.
3. Publish with consistent `meta` keys producers and filters agree on.
4. Validate delivery with a synthetic publish before relying on filters in production.
