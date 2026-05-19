<!-- canonical-for: EVENTS_AND_ACTIONS_FILTERS -->
<!-- used-by: -->

# Filters: Basic Filters and Advanced JSONPath

The canonical reference for narrowing which events trigger an Events & Actions listener.

## Three Filter Types

| Type | When to use |
|---|---|
| **No filter** | Trigger on every matching event from this source |
| **Basic filters** | UI-driven simple match on Channel or Sender ID |
| **Advanced JSONPath** | Anything more complex — payload fields, metadata, regex |

## Basic Filters

Configured entirely in the Admin Portal UI:

| Field | Options | Value |
|---|---|---|
| Filters | Channel, Sender ID | — |
| Condition | Exact Match, Contains | — |
| Value | Free text | The string to match |

Examples:

| Filter | Condition | Value | Effect |
|---|---|---|---|
| Channel | Exact Match | `room-42` | Only events on `room-42` |
| Channel | Contains | `room-` | Events on any channel containing `room-` |
| Sender ID | Exact Match | `bot-system` | Only events published by `bot-system` |

Use Basic Filters when one criterion is sufficient. For multi-condition or payload-content filtering, use JSONPath.

## Advanced JSONPath

JSONPath is evaluated against the entire event payload. Returns matches → action triggers; no matches → action skips.

### Payload Shape (for Message Sent events)

```json
{
  "channel": "chat-room",
  "uuid": "user-123",
  "message": { /* user payload */ },
  "meta": { /* metadata param */ }
}
```

(`uuid` here is the [PubNub userId/UUID](../../pubnub-app-developer/references/sdk-patterns.md) of the publisher.)

JSONPath expressions reference these top-level fields.

### Common Patterns

**Exact channel match:**
```jsonpath
$.[?(@.channel == 'room-42')]
```

**Channel name regex:**
```jsonpath
$.[?(@.channel =~ /.*my_channel.*/i)]
```

The `=~` operator + `/.../i` regex literal allows case-insensitive matching.

**Field in message payload:**
```jsonpath
$.[?(@.message['some_property'] == 'some_value')]
```

**Nested field:**
```jsonpath
$.[?(@.message.payload.text == 'hello')]
```

**Field in message metadata (the `meta` param to publish):**
```jsonpath
$.[?(@.meta.sensor in ['warn', 'alert'])]
```

**Numeric comparison:**
```jsonpath
$.[?(@.meta.sensor_reading > 25)]
```

**Negation:**
```jsonpath
$.[?(@.message.status != 'error')]
```

**Combination (AND):**
```jsonpath
$.[?(@.channel =~ /.*some_suffix/i && @.meta.sensor_reading > 25 && @.message.status != 'error')]
```

**Combination (OR):**
```jsonpath
$.[?(@.message.priority == 'high' || @.message.urgency == 'urgent')]
```

### Operators Cheat Sheet

| Operator | Meaning |
|---|---|
| `==`, `!=` | Equality / inequality |
| `<`, `<=`, `>`, `>=` | Numeric comparison |
| `=~` | Regex match (with `/regex/flags` literal) |
| `in [...]` | Value is in the array literal |
| `nin [...]` | Value is NOT in the array literal |
| `&&` | AND |
| `\|\|` | OR |
| `!` | NOT (e.g., `!(@.muted == true)`) |

### Test Your JSONPath Before Saving

Use [http://jsonpath.com](http://jsonpath.com) or `jp.py` (Python `jsonpath-ng`) with a captured payload sample. **A wrong JSONPath silently drops events** — there is no error, the listener simply never triggers.

```bash
# Quick local test with jq if you have a sample payload
cat sample.json | jq '. | select(.channel == "room-42")'
```

For the actual JSONPath evaluator semantics on PubNub, the canonical test is the live event flow itself.

## Schema-Versioned Filtering

If your messages carry a [`schema_version` field in the envelope](../../pubnub-reliability/references/schema-versioning.md), use it to scope the listener to a specific message format:

```jsonpath
$.[?(@.message.schema_version == 2)]
```

This pairs naturally with version-coordinated rollouts.

## Filter Examples by Use Case

### Only High-Priority Chat Messages

```jsonpath
$.[?(@.channel =~ /^chat\./i && @.message.priority == 'high')]
```

### Only Telemetry Above a Threshold

```jsonpath
$.[?(@.channel == 'sensors' && @.meta.reading > 80)]
```

### Only Messages from a Specific Bot

```jsonpath
$.[?(@.uuid == 'bot-moderator-1')]
```

### Exclude Internal System Messages

```jsonpath
$.[?(!(@.message.system == true))]
```

### Multi-Tenant: Only Tenant 42

```jsonpath
$.[?(@.channel =~ /^t42\./i)]
```

## Filter Performance

JSONPath filters run in PubNub's E&A engine before the action fires. They are fast but not free.

- Prefer **Basic Filters** when they suffice — slightly faster.
- Avoid backtracking-heavy regex (`.*` repeated, lookahead) — use simple regex.
- Combine many criteria into a single JSONPath rather than many narrow listeners.

## Common Anti-Patterns

| Anti-pattern | Fix |
|---|---|
| Saving JSONPath without testing → silent drop | Test on a captured payload sample first |
| Trying to filter on payload size or counts | Not supported in JSONPath; do that in your receiver |
| One listener per channel for hundreds of channels | One listener with a regex JSONPath |
| Forgetting the `?(...)` filter syntax around predicates | Without `?(...)` it's not a filter expression |
| Case-sensitive regex when channels mix cases | Add `/i` flag |
| Trusting unknown fields exist | Use `&&` with existence checks if data shape varies |

## Related Reading

- [event-types.md](event-types.md) — payload shapes by event type
- [action-targets.md](action-targets.md) — what runs when filter passes
- [retries-and-batching.md](retries-and-batching.md) — what happens after action runs
