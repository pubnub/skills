<!-- canonical-for: ILLUMINATE_DECISIONS -->
<!-- used-by: -->

# Illuminate Decisions: The 4-Step Workflow

The canonical reference for Illuminate Decisions, **the most error-prone Illuminate API**. This document captures every gotcha so the agent does not hit the same 500 errors twice.

## What a Decision Is

A Decision evaluates conditions against a Metric, Business Object, or Query and automatically fires actions (publish a message, call a webhook, update App Context metadata) when rules match.

## Source Types

A Decision has exactly one `sourceType`. The shape of `inputFields` depends entirely on the source type:

| `sourceType` | `inputFields` pattern | `executionFrequency` |
|---|---|---|
| `METRIC` — aggregation (SUM/AVG/MIN/MAX) | One `MEASURE` field (`sourceId` = metric's `measureId`) + one `DIMENSION` per dimension field | **Required** |
| `METRIC` — count (COUNT/COUNT_DISTINCT) | One `BUSINESSOBJECT` field (`sourceId` = BO ID, `dataType: "NUMERIC"`) + one `DIMENSION` per dimension field | **Required** |
| `BUSINESSOBJECT` | One `FIELD` per BO field to evaluate (`sourceId` = BO field ID) | **Omit entirely — do not send `null`** |
| `QUERY` | One `QUERYFIELD` per query output field (`sourceId` from `GET /v2/queries/{id}/fields`) | **Required** |

## Required Fields on Every Decision POST

The following fields are **required** despite being documented as optional. Omitting any of them returns `500 Internal Server Error` (not a descriptive 400):

| Field | Required value |
|---|---|
| `activeFrom` | ISO 8601 UTC string, e.g. `"2026-01-01T00:00:00Z"` |
| `activeUntil` | ISO 8601 UTC string, e.g. `"2027-12-31T23:59:59Z"` |
| `hitType` | Always `"SINGLE"` |
| `executeOnce` | Always `false` |

## inputField Schema

```json
{
  "name": "Human-readable name",
  "sourceType": "FIELD | MEASURE | DIMENSION | BUSINESSOBJECT | QUERYFIELD",
  "sourceId": "<id>",
  "dataType": "NUMERIC | TEXT",
  "order": 1
}
```

- `order` controls column display in the Illuminate UI.
- `dataType` is `NUMERIC` for aggregation/count input fields, `TEXT` for dimension fields.
- **Omit `id` on POST — it is auto-generated.**

### Example: COUNT Metric Decision inputFields

```json
"inputFields": [
  { "name": "Count of My Events", "sourceType": "BUSINESSOBJECT", "sourceId": "<bo-id>",      "dataType": "NUMERIC", "order": 1 },
  { "name": "User ID",             "sourceType": "DIMENSION",      "sourceId": "<user-field>", "dataType": "TEXT",    "order": 2 },
  { "name": "Event Type",          "sourceType": "DIMENSION",      "sourceId": "<type-field>", "dataType": "TEXT",    "order": 3 }
]
```

## Action Types

| `actionType` | What it does |
|---|---|
| `PUBNUB_PUBLISH` | Publishes a message to a PubNub channel (requires valid pub + sub key in template) |
| `WEBHOOK_EXECUTION` | POSTs a payload to an external URL |
| `APPCONTEXT_SET_USER_METADATA` | Sets custom fields on a user object |
| `APPCONTEXT_SET_CHANNEL_METADATA` | Sets custom fields on a channel object |
| `APPCONTEXT_SET_MEMBERSHIP_METADATA` | Sets custom fields on a user-channel membership |

For App Context action targets, see [pubnub-app-context/references/users.md](../../pubnub-app-context/references/users.md). For PUBNUB_PUBLISH, the messages flow through the standard [publish/subscribe pipeline](../../pubnub-app-developer/references/publish-subscribe.md) (which also explains [`userId` / UUID](../../pubnub-app-developer/references/sdk-patterns.md) semantics). For richer event-driven action targets like webhook/Lambda/Kafka/SQS, prefer [pubnub-events-and-actions](../../pubnub-events-and-actions/references/event-types.md) instead of WEBHOOK_EXECUTION when no threshold logic is needed.

**Critical naming**: action objects use `"actionType"` (not `"type"`). `outputFields` use `"variable"` and `"name"` (not `"type"`).

## Condition Operations

```
ANY
NUMERIC_EQUALS, NUMERIC_NOT_EQUALS
NUMERIC_GREATER_THAN, NUMERIC_GREATER_THAN_EQUALS
NUMERIC_LESS_THAN, NUMERIC_LESS_THAN_EQUALS
NUMERIC_INCLUSIVE_BETWEEN  (argument: "low,high")
STRING_EQUALS, STRING_NOT_EQUALS
STRING_CONTAINS
```

Use `ANY` for "don't-care" input fields in a rule that still must include them in `inputValues`.

## Avoiding False Positives (a Decision That Fires Every Window)

A Decision can be structurally valid yet fire on **every** evaluation window regardless of real activity. This is one of the most common "it's wired but wrong" problems, and it has two usual causes:

1. **An unfiltered measure Metric.** If the source Metric (`AVG`/`SUM`/`MIN`/`MAX`) is not scoped to one event/message type, it aggregates across **every** record on the channel. Records that don't carry the measured field contribute nothing, the aggregate is meaningless, and on a quiet window it collapses toward `0`. Scope the Metric with a filter so it sees only the relevant records — see [metrics.md](metrics.md), "Filters Scope the Metric."

2. **A `NUMERIC_LESS_THAN` threshold on a sparse window.** When few or no matching records arrive in a window, the aggregate drops to/near `0`, so a `... LESS_THAN X` rule is satisfied (`0 < X`) and the action fires — even though nothing really happened. Prefer **`NUMERIC_GREATER_THAN`** with the threshold set **above the metric's normal baseline**: an empty window reads as `0`, which a `> X` rule does *not* satisfy, so the Decision fires only on a genuine spike. If "the value fell below X" is truly the intent, make sure the window stays populated (steady traffic, or a longer `evaluationWindow`) so an idle window isn't misread as a breach.

**Rule of thumb: filter the source Metric to one event type, and prefer "greater-than-baseline" thresholds.** Together these eliminate the large majority of constantly-firing Decisions.

## Execution Limit Types

Valid values for `executionLimitType` in `actionValues`:

```
ALWAYS
ONCE
ONCE_PER_INTERVAL
ONCE_PER_CONDITION_GROUP
ONCE_PER_INTERVAL_PER_CONDITION_GROUP
```

> **`ONCE_PER_INTERVAL_PER_CONDITION` is NOT valid.** It appears in some docs but the API rejects it. Use `ONCE_PER_INTERVAL_PER_CONDITION_GROUP` instead.

## The 4-Step Creation Workflow

Decisions cannot be created in a single call because rules need to reference auto-generated IDs from `inputFields`, `outputFields`, and `actions`.

### Step 1: POST with `enabled: false` and `rules: []`

Include all `inputFields`, `outputFields`, and `actions` **without `id` fields** — they are auto-generated.

```bash
curl -X POST https://admin-api.pubnub.com/v2/illuminate/decisions \
  -H "Authorization: $ILLUMINATE_API_KEY" \
  -H "PubNub-Version: 2026-02-09" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "High-Volume User Mute",
    "sourceType": "METRIC",
    "sourceId": "<metric-id>",
    "executionFrequency": 60,
    "activeFrom":  "2026-01-01T00:00:00Z",
    "activeUntil": "2027-12-31T23:59:59Z",
    "hitType": "SINGLE",
    "executeOnce": false,
    "enabled": false,
    "rules": [],
    "inputFields": [
      { "name": "Message Count", "sourceType": "BUSINESSOBJECT", "sourceId": "<bo-id>",      "dataType": "NUMERIC", "order": 1 },
      { "name": "User ID",       "sourceType": "DIMENSION",      "sourceId": "<user-field>", "dataType": "TEXT",    "order": 2 }
    ],
    "outputFields": [
      { "name": "Muted User", "variable": "mutedUser" }
    ],
    "actions": [
      {
        "actionType": "APPCONTEXT_SET_USER_METADATA",
        "name": "Mute the user",
        "actionValues": {
          "executionLimitType": "ONCE_PER_INTERVAL_PER_CONDITION_GROUP",
          "executionInterval": 3600,
          "userIdTemplate": "${User ID}",
          "customMetadata": { "muted": true, "muted_reason": "high_volume", "muted_at": "${timestamp}" }
        }
      }
    ]
  }'
```

### Step 2: Read the Response — Extract Auto-Generated IDs

The response contains the auto-generated `id` for every `inputFields[*]`, `outputFields[*]`, and `actions[*]`. Save them.

### Step 3: PUT with Rules

`rules` must reference the IDs returned by step 2. **Every input field MUST appear in every rule's `inputValues`** — use `operation: "ANY"` for don't-care fields.

```bash
curl -X PUT https://admin-api.pubnub.com/v2/illuminate/decisions/<decision-id> \
  -H "Authorization: $ILLUMINATE_API_KEY" \
  -H "PubNub-Version: 2026-02-09" \
  -H "Content-Type: application/json" \
  -d '{
    ...full body from step 1, with auto-generated ids filled in...,
    "rules": [
      {
        "name": "Mute users sending >100 msgs/min",
        "conditionGroups": [
          {
            "conditions": [
              { "inputFieldId": "<count-input-id>", "operation": "NUMERIC_GREATER_THAN", "value": "100" }
            ]
          }
        ],
        "inputValues": [
          { "inputFieldId": "<count-input-id>", "operation": "NUMERIC_GREATER_THAN", "value": "100" },
          { "inputFieldId": "<user-input-id>",  "operation": "ANY",                    "value": "" }
        ],
        "actionIds": ["<action-id>"]
      }
    ]
  }'
```

### Step 4: PUT with `enabled: true`

```bash
curl -X PUT https://admin-api.pubnub.com/v2/illuminate/decisions/<decision-id> \
  -H "Authorization: $ILLUMINATE_API_KEY" \
  -H "PubNub-Version: 2026-02-09" \
  -H "Content-Type: application/json" \
  -d '{ ...full body..., "enabled": true }'
```

## Decision Body Rules by Source Type

These are the per-source-type rules that, if violated, return errors:

### BUSINESSOBJECT decisions

- **Require both `"sourceId": "<bo-id>"` and `"businessObjectId": "<bo-id>"`** at the top level (set to the same BO ID). Omitting `sourceId` returns `400: "sourceId must be a UUID"`.
- `inputFields` must use `"sourceType": "FIELD"` — **not** `"DIMENSION"`.
- **Completely omit `executionFrequency` from PUT bodies** — sending `null` triggers a plan error.

### METRIC decisions (COUNT/COUNT_DISTINCT)

- Use `"sourceType": "BUSINESSOBJECT"` with `sourceId` = the BO ID for the count input field. (There is no `measureId` for COUNT metrics.)
- Use `"sourceType": "DIMENSION"` for each grouped dimension input field.

### METRIC decisions (SUM/AVG/MIN/MAX)

- Use `"sourceType": "MEASURE"` with `sourceId` = the metric's `measureId` for the aggregated value.
- Use `"sourceType": "DIMENSION"` for each grouped dimension input field.

### QUERY decisions

- Use `"sourceType": "QUERYFIELD"` for all input fields.
- Get `sourceId` values from `GET /v2/queries/{id}/fields`.

## Action and Output Field Format Pitfalls

- Action objects use `"actionType"` (not `"type"`). E.g., `"actionType": "WEBHOOK_EXECUTION"`.
- `outputFields` require a `"variable"` (camelCase identifier used in `${variable}` template references) and a `"name"` field. **No `"type"` field.**
- Template variables support `${outputVariable}` and `${inputFieldName}` syntax. When using the `manage_illuminate` MCP tool's `create` operation, `${...}` references in action template bodies are automatically resolved from human-readable names to auto-generated UUIDs — use `${User ID}`, `${points}`, etc., instead of raw UUIDs.
- `PUBNUB_PUBLISH` actions require a valid publish key and subscribe key in the template — see [pubnub-keyset-management/references/keysets-and-environments.md](../../pubnub-keyset-management/references/keysets-and-environments.md). The target **channel** must also be a valid PubNub channel name: an invalid channel makes the action log as *unsuccessful* and the message is never delivered, even though the rule matched. In particular, keep the channel to **≤ 3 dot-delimited levels** (`a.b.c`) — a deeper channel like `a.b.c.d` has been observed to fail delivery — and avoid reserved characters.

## Updating an Active Decision

Active Decisions are **immutable for structural fields** (`inputFields`, `outputFields`, `actions`). To change them:

1. PUT with `enabled: false`.
2. PUT with the structural changes.
3. PUT with `enabled: true`.

PUT is always **full replacement** — include the complete body every time.

## Account Limits

| Decision type | Limit | Error returned |
|---|---|---|
| `METRIC` decisions | 3 per account | `400: "A business object cannot have more than 3 associated decisions."` |
| `QUERY` decisions | ~10–11 per account | `500 Internal Server Error` (no descriptive message) |
| `BUSINESSOBJECT` decisions | No enforced limit | — |

**Before creating a new METRIC or QUERY decision, list existing decisions of that type.** If at or near the limit, ask the user which existing decision to delete before retrying. **Never delete without explicit confirmation.**

## Common Error Decoder Ring

| Error | Cause | Fix |
|---|---|---|
| `500 Internal Server Error` on POST | Missing `activeFrom`, `activeUntil`, `hitType`, or `executeOnce` | Add all four |
| `400: "Number of inputValues (N) does not match number of inputFields (M)"` | A rule's `inputValues` doesn't include every `inputField` | Add an entry (use `operation: "ANY"` for don't-care) for every input field |
| `400: "sourceId must be a UUID"` on BUSINESSOBJECT decision | Set `businessObjectId` only, not also `sourceId` | Set BOTH to the same BO ID |
| Plan error on BUSINESSOBJECT PUT | Sent `executionFrequency: null` | Omit the field entirely |
| Action does nothing | Used `"type"` instead of `"actionType"` | Rename to `actionType` |
| Output field rejected | Used `"type"` instead of `"variable"` | Use `variable` and `name` only |
| `executionLimitType` rejected | Used `ONCE_PER_INTERVAL_PER_CONDITION` | Use `ONCE_PER_INTERVAL_PER_CONDITION_GROUP` |
| `400: "A business object cannot have more than 3 associated decisions."` | Hit METRIC decision limit | Delete one (with user confirmation) and retry |
| Decision fires every evaluation window regardless of real data | Source Metric not filtered to one event type (aggregates across all messages → collapses toward `0`), and/or a `NUMERIC_LESS_THAN` threshold an idle window satisfies | Filter the source Metric ([metrics.md](metrics.md)); prefer `NUMERIC_GREATER_THAN` above baseline — see "Avoiding False Positives" |
| Action shows in the log but the message never arrives | `PUBNUB_PUBLISH` target channel is not a valid PubNub channel name | Use a simple channel name (≤ 3 dot-delimited levels, no reserved characters) |

## Related Reading

- [service-integration-auth.md](service-integration-auth.md) — auth setup
- [business-objects.md](business-objects.md) — must exist before any Decision
- [metrics.md](metrics.md) — usually the source for a Decision
- [queries-adhoc-vs-saved.md](queries-adhoc-vs-saved.md) — Queries as Decision sources
