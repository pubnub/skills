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
| `message_id` | [UUID v4](../../pubnub-app-developer/SKILL.md) for [idempotent publish](idempotent-publish.md) + [dedup](dedup-on-merge.md). |
| `sent_at` | Producer timestamp in ms. Useful for stale-message detection. |
| `type` | Discriminator if the payload shape varies by event type. |
| `payload` | The application-specific body. |

For [logging correlation](../../pubnub-observability/references/logging-correlation.md) include `schema_version` and `message_id` in every log line.

## Schema Version + PubNub History

**Key PubNub-specific concern:** Old messages in [Message Persistence](../../pubnub-history/references/pagination-and-ordering.md) keep their original `schema_version`. When you fetch history months later, you'll see a mix of versions. Your receiver must handle multiple schema versions when replaying history.

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

## Schema Version + PubNub Functions

When writing [PubNub Functions](../../pubnub-functions/SKILL.md), check `schema_version` before processing the message. A Function written for v1 should not silently mishandle v2 messages.

## Schema Version + Illuminate

If you use [Illuminate Business Objects](../../pubnub-illuminate/references/business-objects.md), include `schema_version` as a Business Object field. Then [scope Metrics](../../pubnub-illuminate/references/metrics.md) by version when migrating, and add Decision rules that detect version drift.

## Schema Version + App Context

[App Context custom fields](../../pubnub-app-context/references/users.md) also evolve. Store a `schema_version` field in `custom` so readers know what shape to expect.

## Related Reading

- [idempotent-publish.md](idempotent-publish.md) — envelope also carries `message_id`
- [dedup-on-merge.md](dedup-on-merge.md) — dedup keys live in the envelope
- [pubnub-app-developer/SKILL.md](../../pubnub-app-developer/SKILL.md) — what publish/subscribe actually transmits
- [pubnub-functions/SKILL.md](../../pubnub-functions/SKILL.md) — Functions must handle schema evolution
- [pubnub-illuminate/references/business-objects.md](../../pubnub-illuminate/references/business-objects.md) — Illuminate field extraction depends on stable shape
