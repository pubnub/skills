<!-- canonical-for: ILLUMINATE_DASHBOARDS -->
<!-- used-by: -->

# Illuminate Dashboards

The canonical reference for Illuminate Dashboards: chart types, sizes, date ranges, and the full-replacement PUT behavior.

## What a Dashboard Is

A Dashboard groups one or more charts. Each chart displays a Metric over time. Charts can overlay Decision trigger events so you can visually correlate rule firings with metric spikes.

## Chart Types

| `viewType` | When to use |
|---|---|
| `LINE` | Time-series line chart — trend monitoring |
| `BAR` | Bar chart — comparing dimensions at a point in time |
| `STACKED` | Stacked bar — composition breakdowns over dimensions |

## Chart Size

Each chart's `size`:

- `FULL` — full-width row
- `HALF` — half-width, two charts per row

## Date Ranges

Allowed values:

```
30 minutes
1 hour
24 hours
3 days
1 week
30 days
3 months
Custom date     (requires startDate and endDate in ISO 8601)
```

Other values are rejected.

## Creating a Dashboard

```bash
curl -X POST https://admin-api.pubnub.com/v2/illuminate/dashboards \
  -H "Authorization: $ILLUMINATE_API_KEY" \
  -H "PubNub-Version: 2026-02-09" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Chat Moderation Overview",
    "dateRange": "24 hours",
    "charts": [
      {
        "name": "Messages per Minute",
        "metricId": "<msgs-per-minute-metric-id>",
        "viewType": "LINE",
        "size": "FULL",
        "decisionIds": ["<high-volume-mute-decision-id>"]
      },
      {
        "name": "Top Posters",
        "metricId": "<msgs-per-user-metric-id>",
        "viewType": "BAR",
        "size": "HALF",
        "dimensionIds": ["<user-id-dimension-field-id>"]
      },
      {
        "name": "Channel Composition",
        "metricId": "<msgs-per-channel-metric-id>",
        "viewType": "STACKED",
        "size": "HALF",
        "dimensionIds": ["<channel-dimension-field-id>"]
      }
    ]
  }'
```

## Updating a Dashboard

> **The `charts` array is a FULL REPLACEMENT on PUT.** Always include all existing charts (with their `charts[].id`) to avoid accidental deletion.

```bash
# Read the current dashboard first
curl -X GET https://admin-api.pubnub.com/v2/illuminate/dashboards/<dashboard-id> \
  -H "Authorization: $ILLUMINATE_API_KEY" \
  -H "PubNub-Version: 2026-02-09"

# Then PUT the FULL body, with the existing chart ids preserved + the new chart appended
curl -X PUT https://admin-api.pubnub.com/v2/illuminate/dashboards/<dashboard-id> \
  -H "Authorization: $ILLUMINATE_API_KEY" \
  -H "PubNub-Version: 2026-02-09" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Chat Moderation Overview",
    "dateRange": "24 hours",
    "charts": [
      { "id": "<existing-chart-1-id>", ...existing chart 1 body... },
      { "id": "<existing-chart-2-id>", ...existing chart 2 body... },
      { "id": "<existing-chart-3-id>", ...existing chart 3 body... },
      { ...new chart body, no id... }
    ]
  }'
```

If you PUT a `charts` array that omits an existing chart by id, that chart is **deleted**. This is the most common Dashboard footgun.

## Decision Overlays

Add `decisionIds` to a chart to overlay Decision trigger events on top of the metric. Triggers appear as markers on the time axis, letting you visually verify that the Decision fires when expected.

```json
{ "name": "Messages per Minute",
  "metricId": "<m1>",
  "viewType": "LINE",
  "size": "FULL",
  "decisionIds": ["<d1>", "<d2>"] }
```

Useful during Decision development to confirm thresholds are sane.

## Dimension Breakdowns

Add `dimensionIds` to break down a metric by an underlying Business Object dimension. For a `messages-per-user` metric, setting `dimensionIds` to `[user-id-field]` produces a per-user breakdown.

Limit dimension cardinality — a chart broken down by a dimension with thousands of values is unreadable and expensive to render.

## Custom Date Range

```json
{
  "dateRange": "Custom date",
  "startDate": "2026-04-29T00:00:00Z",
  "endDate":   "2026-04-29T23:59:59Z"
}
```

`startDate` and `endDate` are required when `dateRange` is `"Custom date"`.

## Deleting a Dashboard

Deleting a Dashboard removes only the visualization. Underlying Metrics and Decisions are not affected.

```bash
curl -X DELETE https://admin-api.pubnub.com/v2/illuminate/dashboards/<dashboard-id> \
  -H "Authorization: $ILLUMINATE_API_KEY" \
  -H "PubNub-Version: 2026-02-09"
```

This is the **only Illuminate delete that does not cascade**. Safe to run after user confirmation.

## Best Practices

- **Overlay `decisionIds`** to visually confirm rules fire on expected metric movements.
- **Use `dimensionIds` for the most operationally relevant breakdown** (user, region, event type) — not every available dimension.
- **Always GET before PUT** so you preserve existing chart ids.
- **Name dashboards by audience** (`On-Call`, `Moderation`, `Engagement KPIs`) so finding the right one is fast.
- **One dashboard per concern** — resist the temptation to put 12 charts on one page.

## Common Errors

| Error | Cause | Fix |
|---|---|---|
| Existing chart disappears after PUT | PUT body omitted that chart's `id` | Always include all existing chart ids |
| `400: invalid dateRange` | Used a non-allowed range | Use one of the listed values |
| `400: startDate required` | `dateRange: "Custom date"` without `startDate`/`endDate` | Add both |
| Chart shows no data | Underlying Metric has no data, or Metric was disabled | Verify the Metric is active and has matching events |

## Related Reading

- [metrics.md](metrics.md) — chart data source
- [queries-adhoc-vs-saved.md](queries-adhoc-vs-saved.md) — alternative chart data source
- [decisions-4-step-workflow.md](decisions-4-step-workflow.md) — overlay sources
