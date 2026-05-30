<!-- canonical-for: ILLUMINATE_METRICS -->
<!-- used-by: -->

# Illuminate Metrics

The canonical reference for Illuminate Metrics: aggregations, evaluation windows, dimensions vs measures, filter scoping.

## What a Metric Is

A Metric defines an aggregation applied to a Business Object field over a fixed time window. Metrics are the data source for:

- Threshold-based Decisions
- Dashboard charts

Each Metric is scoped to exactly one Business Object.

## Aggregation Functions

| Function | `measureId` required? | Field type required |
|---|---|---|
| `COUNT` | No | Any |
| `COUNT_DISTINCT` | No | Any (counts unique values) |
| `AVG` | Yes | `NUMERIC` |
| `SUM` | Yes | `NUMERIC` |
| `MIN` | Yes | `NUMERIC` |
| `MAX` | Yes | `NUMERIC` |

**Why this matters for Decisions**: COUNT and COUNT_DISTINCT have no `measureId`, so when wiring them into a Decision's `inputFields`, you use `sourceType: "BUSINESSOBJECT"` (with the BO id). For SUM/AVG/MIN/MAX, you use `sourceType: "MEASURE"` (with the metric's `measureId`). See [decisions-4-step-workflow.md](decisions-4-step-workflow.md).

## Evaluation Windows

Only these values (in seconds) are accepted:

```
60, 300, 600, 900, 1800, 3600, 86400
```

That's: 1 minute, 5 minutes, 10 minutes, 15 minutes, 30 minutes, 1 hour, 1 day.

Other values are rejected. Pick the smallest window that still gives meaningful aggregation for your alert sensitivity.

## Dimensions vs Measures

| Concept | Type | Purpose | Field name in API |
|---|---|---|---|
| **Dimension** | `TEXT` | Group-by axis (per user, per region, per event type) | `dimensionIds: [...]` |
| **Measure** | `NUMERIC` | The value being aggregated | `measureId: <field-id>` |

A `COUNT` metric has only dimensions (it's counting matching rows per dimension combination). A `SUM(amount)` metric has both a measure (`amount`) and dimensions (e.g., per `user_id`).

## Creating a Metric

### COUNT Example: Messages per User per Channel per Minute

```bash
curl -X POST https://admin-api.pubnub.com/v2/illuminate/metrics \
  -H "Authorization: $ILLUMINATE_API_KEY" \
  -H "PubNub-Version: 2026-02-09" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Messages per User per Channel (1m)",
    "businessObjectId": "<bo-id>",
    "function": "COUNT",
    "evaluationWindow": 60,
    "dimensionIds": ["<user-id-field-id>", "<channel-field-id>"],
    "filters": []
  }'
```

### SUM Example: Total Spending per User per Hour

```json
{
  "name": "Spending per User (1h)",
  "businessObjectId": "<bo-id>",
  "function": "SUM",
  "evaluationWindow": 3600,
  "measureId": "<amount-field-id>",
  "dimensionIds": ["<user-id-field-id>"],
  "filters": [
    { "fieldId": "<event-type-field-id>", "operation": "STRING_EQUALS", "value": "purchase" }
  ]
}
```

### Filters Scope the Metric

Use `filters` to scope a Metric to relevant events only. Without filters, the Metric counts every record matching the schema. With filters, the Metric only includes rows where each filter expression matches.

**For measure Metrics (`AVG`/`SUM`/`MIN`/`MAX`) this is a correctness requirement, not just a cost optimization.** A single channel usually carries several message/event types, and the Business Object ingests **all** of them (see [business-objects.md](business-objects.md)). An unfiltered measure Metric therefore aggregates across every type; records that don't contain the measured field contribute nothing, the aggregate becomes meaningless, and on a quiet window it collapses toward `0`. A threshold Decision built on such a Metric then misfires (typically firing on **every** window — see [decisions-4-step-workflow.md](decisions-4-step-workflow.md), "Avoiding False Positives"). **Always filter a measure Metric to the single event type that carries the measure**, e.g. `event STRING_EQUALS "purchase"`.

```json
"filters": [
  { "fieldId": "<event-type-field-id>", "operation": "STRING_EQUALS", "value": "purchase" },
  { "fieldId": "<region-field-id>", "operation": "STRING_EQUALS", "value": "us-west" }
]
```

Multiple filters combine with AND.

## Updating a Metric

You **cannot update a Metric while a referencing Decision is enabled**. The required sequence:

1. Disable all Decisions that reference this Metric (`enabled: false`).
2. Update the Metric.
3. Re-enable the Decisions.

## Cascade Delete

Deleting a Metric **permanently deletes all Decisions that reference it.** Confirm with the user before deleting.

## Account Limits

- 3 METRIC-source Decisions per account total. (See [decisions-4-step-workflow.md](decisions-4-step-workflow.md) for the full table.)
- No enforced cap on the number of Metrics themselves, but each contributes to the cost of the keyset.

## Best Practices

- **Add filters early.** A Metric without filters captures every record, which inflates dimension cardinality and dashboard cost.
- **Pick the smallest useful evaluation window.** A 60-second window with high cardinality is expensive — go to 300 or 900 if minute-level granularity isn't required.
- **Limit dimension cardinality.** A dimension that takes thousands of values (e.g., a free-text field) creates an explosion of buckets. Stick to bounded dimensions (user_id, channel, event_type, region).
- **Name metrics with the window in the name** (`Messages per User (1m)`) so dashboard authors don't have to guess.

## Common Errors

| Error | Cause | Fix |
|---|---|---|
| `400: invalid evaluationWindow` | Used a non-allowed window | Use 60, 300, 600, 900, 1800, 3600, or 86400 |
| `400: measureId required` | Used SUM/AVG/MIN/MAX without `measureId` | Add a NUMERIC field and reference its id |
| Cannot update Metric | Referencing Decision is still enabled | Disable Decision first |
| Metric returns no data | Filters too narrow, or dimension never matches | Verify a sample message via [`subscribe_and_receive_pubnub_messages`](../../pubnub-app-developer/references/publish-subscribe.md) |
| A Decision on this Metric fires every window | Measure Metric not scoped to one event type — it averages across all message types and collapses toward `0` | Add a `STRING_EQUALS` filter on the event-type field; see [decisions-4-step-workflow.md](decisions-4-step-workflow.md), "Avoiding False Positives" |

## Related Reading

- [business-objects.md](business-objects.md) — must exist before creating a Metric
- [decisions-4-step-workflow.md](decisions-4-step-workflow.md) — wiring a Metric into a Decision
- [dashboards.md](dashboards.md) — visualizing a Metric
