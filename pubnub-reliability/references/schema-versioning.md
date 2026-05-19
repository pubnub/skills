<!-- canonical-for: SCHEMA_VERSIONING -->
<!-- used-by: pubnub-illuminate, pubnub-app-context, pubnub-app-developer -->

# Schema Versioning of Message Envelopes

The canonical reference for versioning the JSON shape of PubNub messages so old and new clients coexist.

## Why Schema Version

Every PubNub message is a JSON payload that travels through producers, subscribers, history, Functions, and possibly Illuminate. Each of those will evolve at a different rate. Without an explicit schema version, you eventually hit:

- New client crashes on old data shape (no `field_x`)
- Old client misbehaves on new data shape (extra `field_y` is misinterpreted)
- A Function written against shape v1 silently mishandles a shape v2 message
- Illuminate Business Object fields fail to extract because the JSONPath moved

Putting a `schema_version` field in every envelope is the cheapest insurance you can buy.

## The Envelope

```json
{
  "schema_version": 1,
  "message_id": "550e8400-e29b-41d4-a716-446655440000",
  "sent_at": 1714435000123,
  "type": "chat_message",
  "payload": {
    "text": "hello",
    "channel": "room-42"
  }
}
```

Required envelope fields:

| Field | Purpose |
|---|---|
| `schema_version` | Integer. Bumped on any change to the shape. |
| `message_id` | [UUID v4](../../pubnub-app-developer/references/sdk-patterns.md) for [idempotent publish](idempotent-publish.md) + [dedup](dedup-on-merge.md). |
| `sent_at` | Producer timestamp in ms. Useful for stale-message detection. |
| `type` | Discriminator if the payload shape varies by event type. |
| `payload` | The application-specific body. |

## Version Bumping Rules

| Change | Bump? |
|---|---|
| Add an optional field that old clients can ignore | Optional bump |
| Add a required field | **Bump** |
| Rename a field | **Bump** (same as remove + add) |
| Change a field's type | **Bump** |
| Restructure nesting | **Bump** |
| Change semantics of an existing field | **Bump** |

When you bump, **never reuse the previous version number for a different shape**. v1 is forever v1.

## Receiver Strategy: Tolerant Reader

Receivers should:

1. Read `schema_version` first.
2. If version is in a known list, parse with the corresponding handler.
3. If version is below the minimum supported, drop with a log line.
4. If version is above the maximum supported, drop with a log line and suggest an app update.
5. Never crash on unknown fields — the payload may include forward-compat data.

```javascript
const HANDLERS = {
  1: handleV1,
  2: handleV2,
  3: handleV3,
};

const MIN_SUPPORTED = 1;
const MAX_SUPPORTED = 3;

function process(envelope) {
  const v = envelope?.schema_version;

  if (typeof v !== 'number') {
    log('schema.version.missing', { message_id: envelope?.message_id });
    return;
  }
  if (v < MIN_SUPPORTED) {
    log('schema.version.too_old', { v });
    return;
  }
  if (v > MAX_SUPPORTED) {
    log('schema.version.too_new', { v });
    promptUserToUpdateApp();
    return;
  }

  HANDLERS[v](envelope.payload);
}
```

For [logging correlation](../../pubnub-observability/references/logging-correlation.md) include `schema_version` and `message_id` in every log line.

## Producer Strategy: Always Tag

The producer always writes the latest version it knows. Don't gate on receiver capability — receivers manage their own tolerance.

```javascript
const SCHEMA_VERSION = 3;

function createMessage(type, payload) {
  return {
    schema_version: SCHEMA_VERSION,
    message_id: crypto.randomUUID(),
    sent_at: Date.now(),
    type,
    payload
  };
}
```

When the schema bumps, ship the new producer first, then the new receivers. Old receivers will drop new messages with a log; new receivers handle both.

## Migration Flow

For a clean v1 → v2 transition:

```
T0: Producer + receiver both at v1.
T1: Receiver updated to handle BOTH v1 and v2 (idempotent additions, default values).
T2: Producer updated to write v2.
T3: After confidence period, drop v1 handler in receivers.
```

Never go T0 → T2 directly without T1. Old receivers will drop the new messages.

## Field-Level vs Envelope-Level Versioning

Two valid approaches:

| Approach | When |
|---|---|
| **Single envelope version** | Most apps. Simple. Everything bumps together. |
| **Per-payload-type version** (`payload.payload_version`) | Many independent message types in one channel. The envelope is stable; payloads version independently. |

The envelope-level version is simpler and almost always sufficient.

## Schema Version + History

Old messages in [Message Persistence](../../pubnub-history/references/pagination-and-ordering.md) keep their original `schema_version`. When you fetch history months later, you'll see a mix of versions. This is exactly why receivers must be tolerant readers.

```javascript
const result = await pubnub.fetchMessages({ channels: ['chat'], count: 100 });
const messages = result.channels['chat'] || [];

const counts = {};
for (const m of messages) {
  const v = m.message?.schema_version ?? 'unknown';
  counts[v] = (counts[v] || 0) + 1;
}
console.log('Schema version mix in last 100:', counts);
```

## Schema Version + Illuminate

If you use [Illuminate Business Objects](../../pubnub-illuminate/references/business-objects.md), include `schema_version` as a Business Object field. Then [scope Metrics](../../pubnub-illuminate/references/metrics.md) by version when migrating, and add Decision rules that detect version drift. (Distinct from SDK-side [`subscribeFilterExpression` message filters](../../pubnub-app-developer/references/message-filters.md) — those operate on metadata, not envelope contents.)

## Schema Version + App Context

[App Context custom fields](../../pubnub-app-context/references/users.md) also evolve. The same versioning discipline applies — store a `schema_version` field in `custom` so readers know what shape to expect.

## Anti-Patterns

| Anti-pattern | Fix |
|---|---|
| No version field in envelope | Add `schema_version`; even v1 should be tagged |
| Bumping without supporting both versions in receivers first | T1 (dual support) before T2 (producer migrates) |
| Reusing v1 with a different shape | v1 is forever v1; bump to v2 |
| Receivers crashing on unknown fields | Tolerant reader: ignore unknown |
| Dropping messages silently with no log | Always log with `message_id` and version |
| String version like `"1.0.0"` for envelope version | Use a monotonic integer; semver belongs to the app, not the message |

## Related Reading

- [idempotent-publish.md](idempotent-publish.md) — envelope also carries `message_id`
- [dedup-on-merge.md](dedup-on-merge.md) — dedup keys live in the envelope
- [pubnub-app-developer/references/publish-subscribe.md](../../pubnub-app-developer/references/publish-subscribe.md) — what publish/subscribe actually transmits
- [pubnub-illuminate/references/business-objects.md](../../pubnub-illuminate/references/business-objects.md) — Illuminate field extraction depends on stable shape
