<!-- canonical-for: ILLUMINATE_QUERIES -->
<!-- used-by: -->

# Illuminate Queries: Ad-hoc vs Saved Format

The canonical reference for Illuminate Queries — flexible data pipelines over Business Object fields, including the **critical format difference** between the ad-hoc and saved endpoints.

## What a Query Is

Queries define flexible pipelines over Business Object fields, supporting aggregation, filtering, joins, [deduplication](../../pubnub-reliability/references/dedup-on-merge.md), and more. Unlike Metrics (fixed aggregations), Queries give full control over data shaping.

Saved Queries can be used as:

- Decision sources
- Dashboard chart data sources

Use a Query when a single fixed aggregation (Metric) doesn't capture the logic — e.g., joins between two Business Objects, multi-stage pipelines, dedup before counting, etc.

## Pipeline Model

```
Sources → Transforms (optional) → Output
```

| Stage | Purpose |
|---|---|
| **Sources** | Reference a Business Object and select fields with a time range |
| **Transforms** | `aggregate`, `window`, `join`, `dedupe`, `calculate`, `unnest`, `filter_join`. Each transform references a source or earlier transform by `id` |
| **Output** | Specify which stage to return from, which fields to `select`, plus `orderBy`, `limit`, `offset` |

## Pre-Built Templates

The Admin Portal includes four pre-built templates for common use cases:

| Template | Use case |
|---|---|
| Cross-Posting Spam | Detect users posting identical content to multiple channels |
| Chat Flooding Spam | Detect users sending an abnormally high message volume |
| Top N Rankings | Rank the top N users or items by a numeric metric |
| Bottom N Rankings | Rank the bottom N users or items by a numeric metric |

Templates are convenient when they fit your need exactly. For anything custom, use the manual format below.

## **Critical Format Difference: Ad-hoc vs Saved**

The `POST /queries/execute` (ad-hoc) endpoint and `POST /queries` (save) endpoint use **different pipeline formats**. This trips up almost every first-time user.

|  | Ad-hoc (`POST /queries/execute`) | Saved (`POST /queries`) |
|---|---|---|
| Wrapper | Top-level `version` + `pipeline` | `definition: { version, pipeline }` |
| GroupBy transform type | `"groupBy"` | `"aggregate"` |
| Transform input ref | `"from": "src"` | `"input": "src"` |
| GroupBy structure | Direct `groupBy` + `aggregations` fields | Nested under `"aggregate": { groupBy, aggregations }` |
| Function names | Any case | **Lowercase** — `"count"`, `"sum"`, `"avg"` |

### Recommended Workflow

1. **Develop with the ad-hoc endpoint** (`POST /v2/queries/execute`) — fast iteration.
2. **When the logic is right, rewrite to saved format** before calling `POST /queries`.
3. **Verify the saved query** by reading it back and executing it through the Decision wiring.

## Ad-hoc Format Example

```json
{
  "version": "2.0",
  "pipeline": {
    "sources": [
      {
        "id": "src",
        "businessObjectId": "<bo-id>",
        "fields": ["user_id", "channel", "text"],
        "timeRange": { "type": "RELATIVE", "value": "PT5M" }
      }
    ],
    "transforms": [
      {
        "id": "grouped",
        "type": "groupBy",
        "from": "src",
        "groupBy": ["user_id"],
        "aggregations": [
          { "field": "channel", "function": "COUNT_DISTINCT", "as": "channels_posted_to" }
        ]
      }
    ],
    "output": {
      "from": "grouped",
      "select": ["user_id", "channels_posted_to"],
      "orderBy": [{ "field": "channels_posted_to", "direction": "DESC" }],
      "limit": 50
    }
  }
}
```

POST to `/v2/queries/execute` to get results immediately.

## Saved Format Example (Same Logic)

```json
{
  "name": "Cross-Channel Posters (Last 5m)",
  "definition": {
    "version": "2.0",
    "pipeline": {
      "sources": [
        {
          "id": "src",
          "businessObjectId": "<bo-id>",
          "fields": ["user_id", "channel", "text"],
          "timeRange": { "type": "RELATIVE", "value": "PT5M" }
        }
      ],
      "transforms": [
        {
          "id": "grouped",
          "type": "aggregate",
          "input": "src",
          "aggregate": {
            "groupBy": ["user_id"],
            "aggregations": [
              { "field": "channel", "function": "count_distinct", "as": "channels_posted_to" }
            ]
          }
        }
      ],
      "output": {
        "from": "grouped",
        "select": ["user_id", "channels_posted_to"],
        "orderBy": [{ "field": "channels_posted_to", "direction": "DESC" }],
        "limit": 50
      }
    }
  }
}
```

POST to `/v2/queries`.

### Side-by-Side Differences in This Example

| Ad-hoc | Saved |
|---|---|
| `{ "version": "2.0", "pipeline": {...} }` | `{ "definition": { "version": "2.0", "pipeline": {...} } }` |
| `"type": "groupBy"` | `"type": "aggregate"` |
| `"from": "src"` | `"input": "src"` |
| Direct `groupBy` and `aggregations` at transform root | Nested under `"aggregate": { groupBy, aggregations }` |
| `"function": "COUNT_DISTINCT"` | `"function": "count_distinct"` |

## Using a Saved Query as a Decision Source

1. Create and save a Query (saved format).
2. Call `GET /v2/queries/{id}/fields` to retrieve the output field IDs the Decision needs.
3. Create a Decision with `sourceType: "QUERY"` and `sourceId: <queryId>`. Use the field IDs from step 2 as `sourceId` values in `inputFields` with `sourceType: "QUERYFIELD"`.

See [decisions-4-step-workflow.md](decisions-4-step-workflow.md) for the full Decision shape.

> **Note**: `GET /queries/{id}/predefined-decisions` only works for **template** queries (Cross-Posting, Chat Flooding, Top N, Bottom N). It returns `404 Not Found` for custom user-created queries — build the Decision manually for those.

## Account Limits

- Maximum **~10 saved Queries per account**. If exceeded, the API returns: `"You have reached the maximum number of queries allowed. Contact PubNub Support to upgrade."`
- Before saving a new Query at this limit, list existing Queries and ask the user which to delete. **Never delete without confirmation.**

## Best Practices

- **Always set `version: "2.0"`** for both ad-hoc and saved.
- **Test ad-hoc first**, then port to saved.
- **Don't change a saved Query's output schema** while a Decision references it — names and types changing breaks the Decision's input field bindings. Disable the Decision first.
- **Deleting a Query** does not delete referencing Dashboards or Decisions, but they may stop functioning correctly. Audit dependents before deleting.

## Common Errors

| Error | Cause | Fix |
|---|---|---|
| `400` on POST `/queries` | Used ad-hoc format (`groupBy`, `from`, uppercase function names) | Rewrite to saved format |
| `404` on GET `/queries/{id}/predefined-decisions` | Custom (non-template) query | Build Decision manually |
| `500` on Decision wiring | Used wrong field IDs (forgot to call `/queries/{id}/fields`) | Get fresh IDs from the endpoint |
| Hit `~10 saved Queries` limit | Account cap | Delete an unused Query (with user confirmation) |

## Related Reading

- [business-objects.md](business-objects.md) — Queries reference Business Object fields
- [metrics.md](metrics.md) — fixed-aggregation alternative
- [decisions-4-step-workflow.md](decisions-4-step-workflow.md) — Queries as Decision sources
- [dashboards.md](dashboards.md) — Queries as Dashboard chart data sources
