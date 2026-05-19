<!-- canonical-for: USAGE_METRICS -->
<!-- used-by: pubnub-keyset-management, pubnub-history -->

# Usage Metrics

The canonical reference for pulling, interpreting, and reconciling PubNub usage / billing metrics.

## Why Pull Usage Metrics

- Reconcile PubNub bills against your own observability data
- Detect cost regressions early (within a billing cycle)
- Capacity-plan for traffic spikes
- Validate cost projections from new features
- Triage cost-related [incidents](incident-runbook.md)

## Transaction Taxonomy

PubNub bills by **transaction type**, not bytes. The categories you'll see in usage metrics:

| Category | Counts |
|---|---|
| Publish | One per `publish()` / `signal()` / `fire()` call |
| Subscribe | One per receive (multiplied by fan-out) |
| Presence | Joins, leaves, and timer ticks |
| History (Storage & Playback) | One per [`fetchMessages`](../../pubnub-history/references/pagination-and-ordering.md) / `history` call (not per message) |
| [App Context](../../pubnub-app-context/references/users.md) (Objects) | One per set/get/delete on user/channel/membership |
| Functions | One per Function execution |
| [Access Manager](../../pubnub-security/references/access-manager.md) | One per `grantToken` |
| Files | One per [`sendFile`](../../pubnub-chat/references/file-sharing.md) / `getFile` (storage cost separate) |
| Mobile Push (FCM/APNs/Huawei) | One per push delivery attempt |

The dominant cost in most apps is **subscribe transactions**, because every publish to a channel with N subscribers produces N receives.

## Pulling Metrics via MCP

Use the `get_pubnub_usage_metrics` MCP tool:

```
get_pubnub_usage_metrics:
  keyset: <keyset-id>
  startDate: 2026-04-01
  endDate: 2026-04-30
  granularity: day | hour
  transactionTypes: [publish, subscribe, presence, ...]
```

Pull per-environment (per-keyset) — see [pubnub-keyset-management/references/keysets-and-environments.md](../../pubnub-keyset-management/references/keysets-and-environments.md). Mixing environments distorts the picture.

## Routine Metrics Review

Run weekly:

| Question | Source |
|---|---|
| Total transactions this week vs last week | Total counts per category |
| Which category grew most? | Per-category trend |
| Any spike days? | Day-by-day series |
| Cost-per-DAU trend | Total transactions / DAU from your own analytics |
| Are we on pace to exceed plan? | Total transactions / days × month length |

## Cost-Per-DAU Calculation

A useful normalized metric:

```
cost_per_dau = (publish_transactions + subscribe_transactions) / daily_active_users
```

Track over time. A rising number with flat DAU means a feature change increased per-user cost. A flat number with rising DAU means linear scaling — usually fine.

## Spike Investigation Workflow

When usage metrics show a spike:

1. **Identify the day and hour.**
2. **Identify the transaction category.** If subscribe spiked, fan-out grew. If publish spiked, publishers grew or coalescing broke. If function spiked, a Function is firing on too many events.
3. **Cross-reference deploys.** What shipped in the spike window?
4. **Cross-reference user behavior.** Did a viral event drive new sessions?
5. **If the spike is gradual over days, check for a leaking subscribe** — clients that subscribe but don't unsubscribe accumulate over time.

For full incident triage see [incident-runbook.md](incident-runbook.md).

## Reconciliation with Your Bill

Monthly:

1. Pull `get_pubnub_usage_metrics` for the billing month.
2. Compare against the invoice line items.
3. Investigate any discrepancy > 5% — usually it's a definitional mismatch (e.g., a category combined differently in billing vs metrics).
4. If there's a real discrepancy, contact PubNub Support with the keyset + date range + your numbers.

## Per-Keyset Alerting

Set up alerts on:

| Trigger | Alert level |
|---|---|
| Daily total transactions > 1.5× rolling 30-day avg | Slack |
| Daily total transactions > 2× rolling 30-day avg | Page |
| Specific category > 3× its rolling avg | Slack |
| Approaching plan limit (> 80% projected) | Email |
| Approaching plan limit (> 95% projected) | Page |

Your monitoring tool (Datadog, etc.) can ingest the metrics and apply these thresholds.

## Capacity Planning

Before a [large event](../../pubnub-scale/references/large-events.md):

1. Pull baseline usage from a comparable past week.
2. Multiply by the projected traffic ratio for the event.
3. Compare to plan limits.
4. If projected total > plan limit, either pre-coordinate with PubNub Support for an event-mode bump or reduce scope.

## Anti-Patterns

| Anti-pattern | Fix |
|---|---|
| Cost spike investigated without categorizing transactions | Always start with per-category breakdown |
| One big shared keyset across envs makes attribution impossible | One keyset per env; metrics are per-keyset |
| No regular review until the bill arrives | Weekly review |
| Comparing total transactions month-over-month without DAU normalization | Use cost-per-DAU |
| Ignoring slow leaks (1% growth per day) | Compare 30-day rolling avg, not just yesterday vs today |
| Not pre-planning for known spikes | Capacity plan with PubNub Support |

## Related Reading

- [cost-and-payload-hygiene.md](cost-and-payload-hygiene.md) — how to reduce costs
- [incident-runbook.md](incident-runbook.md) — triage cost spikes
- [pubnub-keyset-management/references/keysets-and-environments.md](../../pubnub-keyset-management/references/keysets-and-environments.md) — per-env keyset isolation
- [pubnub-scale/references/large-events.md](../../pubnub-scale/references/large-events.md) — pre-event capacity planning
