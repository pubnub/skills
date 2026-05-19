<!-- canonical-for: EVENTS_AND_ACTIONS_DELIVERY -->
<!-- used-by: -->

# Retries, Envelopes, and Batching

The canonical reference for E&A delivery configuration: retry policy, envelope versions, and batching for high-volume targets.

## Retries

E&A uses a **jittered [exponential backoff](../../pubnub-reliability/references/backoff-and-jitter.md)** between attempts. Available on **paid tiers only** — Free has no retry.

### Algorithm

```
delay = random_between(
  minIntervalForRetry = 10s,
  min(maxRetryPeriod = 900s, (baseRetryIntervalDefinedByUser * 2)^attemptNo)
)
```

The cap (`maxRetryPeriod`) is 900 seconds = 15 minutes. The minimum interval is 10 seconds.

### Per-Action Retry Knobs

| Knob | Default | Min | Max |
|---|---|---|---|
| Number of retries | 2 | 1 | 4 |
| Retry interval seconds (`baseRetryIntervalDefinedByUser`) | 450 | 10 | 900 |

For high-priority actions where every event must land, set retries to 4 and accept the longer total delay. For best-effort routing, leave at default.

### Webhook Retry Logic

| HTTP status | Behavior |
|---|---|
| `2xx` | Success, no retry |
| `301` | Follow redirect (up to 3 redirects) |
| `4xx` | Retried per policy (some 4xx may be transient — IAM not yet propagated, etc.) |
| `5xx` | Retried per policy |
| Connection error / timeout | Retried per policy |

### After Final Retry

If all retries fail, the event is permanently lost. There is no dead-letter queue inherent to E&A. Mitigations:

- Make receivers idempotent + tolerant.
- Set a webhook target's "extra metadata on retries" option so retries are distinguishable.
- Pair E&A with PubNub [Message Persistence](../../pubnub-history/references/pagination-and-ordering.md) so you can replay missed events from history.

## Envelope Versions

| Version | Default? |
|---|---|
| 1.0 | No (legacy) |
| 2.0 | No |
| 2.1 | **Yes (default)** |

Each version supports **enveloped** and **unenveloped** formats. Toggle "Is Enveloped" in the UI per action.

### Enveloped vs Unenveloped

- **Unenveloped (default)**: payload is the raw event, e.g. just the message contents and channel.
- **Enveloped**: wrapped with E&A metadata: listener id, channel, timestamp, schema, etc.

### When to Choose Which

| Case | Pick |
|---|---|
| Receiver knows what listener triggered (one listener per receiver) | Unenveloped |
| Receiver handles many listeners and needs context | Enveloped |
| Migrating from a legacy schema | Match the legacy version (1.0 or 2.0) |
| Greenfield integration | 2.1 enveloped or unenveloped per receiver shape |

### Migration Notes

- Run a parallel listener with the new envelope before retiring the old one.
- Test that downstream parsers handle both versions during the transition.
- Update receiver to switch on `schema` field — see [event-types.md](event-types.md).

## Batching

Available on **Webhook** and **Amazon S3** action types.

### Per-Batch Limits

| Knob | Default | Min | Max |
|---|---|---|---|
| Item count | 100 | 1 | 10000 |
| Time bound (seconds) | 5 | 1 | 300 |

A batch is sent when **either** bound is reached first.

### Why Batch

| Target | Why batching matters |
|---|---|
| **S3** | S3 charges per PUT request. 1000 events as 1 batch = 1 PUT; as 1000 individual events = 1000 PUTs. Massive cost difference. |
| **Webhook** | If your receiver does a per-request DB write, batching dramatically reduces DB load. Receiver implementation must handle a `data` array. |

### S3 Multipart Upload

S3 batches over **5 MB** are uploaded as multipart automatically. No extra config needed.

### Webhook Batch Receiver Pattern

```javascript
app.post('/webhook/pubnub', (req, res) => {
  const events = req.body.data || [];     // batch is in `data` array

  for (const event of events) {
    if (seen.has(event.id)) continue;
    seen.add(event.id);
    process(event);
  }

  res.sendStatus(200);                     // ACK once per batch
});
```

The whole batch is one HTTP request. ACK once with `2xx` after processing all items in the batch — partial success isn't supported (PubNub will retry the whole batch).

### Tuning

- For S3 archival: increase batch to 1000 items / 60 seconds. Fewer PUTs, more items per object.
- For latency-sensitive webhooks: decrease batch to 10 items / 1 second. Acceptable cost overhead, low latency.
- For moderate volume webhooks: defaults (100 / 5s) are usually fine.

## Combining Retries + Batching

For batched targets, retry applies to the whole batch. If 1 of 100 items in the batch caused the receiver to return 5xx, the entire batch retries. **Receivers must dedup by event `id`** — see [action-targets.md](action-targets.md).

## Anti-Patterns

| Anti-pattern | Fix |
|---|---|
| Free tier in production with no retry | Upgrade |
| Receiver that returns 4xx for "I've already seen this" | Return 2xx for known duplicates |
| S3 target with no batching | Configure batching to drastically reduce cost |
| Webhook receiver expecting one event per request when batching is on | Iterate `req.body.data` |
| Setting retries to 4 with 900s interval and expecting low latency | Total delay can exceed an hour; lower interval if latency matters |
| Switching envelope version without updating receiver parser | Coordinate update |
| Treating webhook ACK as confirmation each item was processed | ACK is per batch; per-item ack isn't supported |

## Related Reading

- [action-targets.md](action-targets.md) — target-specific behavior
- [event-types.md](event-types.md) — payload schemas
- [filters-and-jsonpath.md](filters-and-jsonpath.md) — narrow what triggers
