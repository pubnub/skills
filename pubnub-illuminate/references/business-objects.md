<!-- canonical-for: ILLUMINATE_BUSINESS_OBJECTS -->
<!-- used-by: -->

# Illuminate Business Objects

The canonical reference for Illuminate Business Objects: schema, field types, JSONPath rules, and the activation sequence.

## What a Business Object Is

A Business Object defines the **schema** for the data Illuminate will ingest from your PubNub channels. Each field maps a JSONPath expression to a typed value extracted from incoming messages.

A Business Object ingests **every** message on the subscribed channel(s) — not just one message or event type. A single channel commonly carries several payload shapes (e.g. `Product_View`, `Add_to_Cart`, `Payment_Processed`), and the BO captures them all; fields whose JSONPath isn't present in a given message are simply empty for that record. This is why downstream **Metrics** that aggregate a field must usually **filter to the one event type that carries it** (see [metrics.md](metrics.md)) — otherwise the aggregate is computed across unrelated messages. A good pattern is to map an explicit type/`event` discriminator field (e.g. `$.message.body.event`) so Metrics and Decisions can filter on it.

It is the **first** Illuminate resource you create. **Metrics**, **Queries**, and **Decisions** depend on it directly; **Dashboards** are account-level and hold **charts** that reference metrics built from Business Object fields (see [Cascade Delete](#cascade-delete) when deleting a BO).

## Message Wrapping

Illuminate wraps every PubNub message it receives as:

```json
{ "message": { "body": <your_pubnub_message> } }
```

All JSONPath expressions in field definitions therefore start with `$.message.body.` — for example `$.message.body.user_id`. Forgetting the prefix is the most common Business Object error.

## Field Types

| Type | Use for | Limit |
|---|---|---|
| `TEXT` | String values; group-by dimensions | 256 characters |
| `TEXT_LONG` | Long string values | 1,000 characters; **max 5 per Business Object** |
| `NUMERIC` | Numbers; **required for AVG, SUM, MIN, MAX metrics** | — |
| `TIMESTAMP` | ISO 8601 timestamps | used in `TIME_DIFF` derived fields |
| `BOOLEAN` | True/false values | — |

Choose `NUMERIC` for any field you might later want to aggregate. Choose `TEXT` for any field you might later want to group by (a dimension).

## Activation Sequence

Business Objects have a strict create-then-activate sequence. Mixing it up returns errors.

### 1. Create with `isActive: false`

Include at least one [subscribe key](../../pubnub-keyset-management/references/keysets-and-environments.md) in the `subkeys` array (Illuminate uses this to know which channel data to ingest). Define all fields. **Omit field `id` on create** — the API auto-generates them.

```bash
curl -X POST https://admin-api.pubnub.com/v2/illuminate/business-objects \
  -H "Authorization: $ILLUMINATE_API_KEY" \
  -H "PubNub-Version: 2026-02-09" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Chat Messages",
    "description": "Tracks chat events for moderation analytics",
    "subkeys": ["sub-c-prod-XXXXXXX"],
    "isActive": false,
    "fields": [
      {
        "name": "user_id",
        "type": "TEXT",
        "jsonPath": "$.message.body.user_id"
      },
      {
        "name": "channel",
        "type": "TEXT",
        "jsonPath": "$.message.body.channel"
      },
      {
        "name": "message_length",
        "type": "NUMERIC",
        "jsonPath": "$.message.body.text.length"
      },
      {
        "name": "is_deleted",
        "type": "BOOLEAN",
        "jsonPath": "$.message.body.deleted"
      },
      {
        "name": "sent_at",
        "type": "TIMESTAMP",
        "jsonPath": "$.message.body.timestamp"
      }
    ]
  }'
```

### 2. Save the IDs from the Response

The response contains the auto-generated `id` for the Business Object and for each field. **Save these.** Metrics, Decisions, and Queries all reference them.

```json
{
  "id": "<bo-id>",
  "fields": [
    { "id": "<user-id-field-id>", "name": "user_id", ... },
    { "id": "<channel-field-id>", "name": "channel", ... },
    ...
  ]
}
```

### 3. Activate by Setting `isActive: true`

```bash
curl -X PUT https://admin-api.pubnub.com/v2/illuminate/business-objects/<bo-id> \
  -H "Authorization: $ILLUMINATE_API_KEY" \
  -H "PubNub-Version: 2026-02-09" \
  -H "Content-Type: application/json" \
  -d '{ ... full body with isActive: true ... }'
```

Once active, Illuminate begins ingesting matching messages.

## Modifying Fields

You **cannot modify fields while a Business Object is active**. The required sequence:

1. Set `isActive: false` (deactivates ingestion immediately).
2. Update the fields.
3. Set `isActive: true` again.

**Deactivate any dependent Decisions first** — they will error if their underlying BO becomes inactive while they're enabled.

## Cascade Delete

Deleting a Business Object **permanently and recursively** deletes:

- All Metrics that reference it
- All Decisions that depend on those Metrics or directly on the BO
- All Queries that reference it

**Dashboards** are not removed wholesale — they are usually account-level. What goes away are **charts** (and similar widgets) that were tied to metrics backed by this Business Object.

The cascade is silent — no confirmation. **Always confirm with the user before any DELETE.**

## Limits

- Max 100 fields per Business Object
- Max 5 `TEXT_LONG` fields per Business Object
- `TEXT` capped at 256 chars per value
- `TEXT_LONG` capped at 1,000 chars per value

## Best Practice Patterns

### Schema Versioning

See [pubnub-reliability/references/schema-versioning.md](../../pubnub-reliability/references/schema-versioning.md) for the canonical pattern. Add an explicit version field to your published messages:

```json
{ "schema_version": 1, "user_id": "...", "channel": "...", ... }
```

Then add a Business Object field that extracts it. This lets you filter Metrics by schema version when migrating.

### Future-Proof Fields

When in doubt, add a field. Adding a field later requires deactivating the BO and any dependent Decisions. Anticipate the dimensions you'll want to group by in Metrics — once shipped, add them up front.

### One BO Per Domain Concept

Don't try to make one BO cover multiple unrelated event types. Create separate Business Objects for `Chat Messages`, `Order Events`, `Game Actions`, etc. They can share the same subscribe key but they have independent schemas.

## Common Errors

| Error | Cause | Fix |
|---|---|---|
| Fields appear empty in Metrics | Missing `$.message.body.` prefix on `jsonPath` | Add the prefix |
| `400` on field create | Field `id` was provided | Omit `id` on create — API generates it |
| `400` on activation | `subkeys` is empty or missing | Include at least one subscribe key |
| Cannot create AVG/SUM Metric | Underlying field is `TEXT`, not `NUMERIC` | Add a `NUMERIC` field; reactivate BO |
| Update fails with "BO is active" | Tried to modify fields while active | Set `isActive: false` first |
| Decisions break after BO update | Forgot to disable Decisions before deactivating | Disable Decisions, then deactivate BO |

## Related Reading

- [service-integration-auth.md](service-integration-auth.md) — auth setup
- [metrics.md](metrics.md) — next step: aggregate Business Object fields
- [decisions-4-step-workflow.md](decisions-4-step-workflow.md) — fire actions based on BO data
