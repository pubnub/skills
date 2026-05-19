<!-- canonical-for: LOGGING_CORRELATION -->
<!-- used-by: pubnub-reliability, pubnub-history -->

# Logging Correlation

The canonical reference for what every PubNub send and receive code path must log so that incidents are diagnosable.

## The Four Required Fields

Log all four on every send and every receive:

| Field | Source | Why |
|---|---|---|
| `channel` | The PubNub channel name | Without it you can't filter logs |
| `message_id` | Client-generated UUID from the envelope | Without it you can't trace a single message across logs |
| `user_id` | The publisher's [`userId`](../../pubnub-app-developer/references/sdk-patterns.md) (and the subscriber's `userId`, separately) | Without it you can't answer "what did this user see?" |
| `timetoken` | Server-assigned, returned by `publish` and present on every received message | Without it you can't correlate to PubNub-side history or billing |

For the `message_id` source see [pubnub-reliability/references/idempotent-publish.md](../../pubnub-reliability/references/idempotent-publish.md).

## Send-Side Logging

For [`addListener` and `pubnub.publish` mechanics](../../pubnub-app-developer/references/publish-subscribe.md) see the canonical owner.

```javascript
async function publishWithLogging(channel, envelope) {
  const ctx = {
    channel,
    message_id: envelope.message_id,
    user_id: pubnub.getUserId(),
    schema_version: envelope.schema_version
  };
  log.info('pubnub.publish.start', ctx);

  try {
    const r = await pubnub.publish({ channel, message: envelope });
    log.info('pubnub.publish.ok', { ...ctx, timetoken: r.timetoken });
    return r;
  } catch (err) {
    log.error('pubnub.publish.error', { ...ctx, error: err.message });
    throw err;
  }
}
```

## Receive-Side Logging

```javascript
pubnub.addListener({
  message: (e) => {
    const ctx = {
      channel: e.channel,
      message_id: e.message?.message_id,
      user_id: pubnub.getUserId(),     // the receiver
      publisher_id: e.publisher,        // the sender
      timetoken: e.timetoken,
      schema_version: e.message?.schema_version
    };
    log.info('pubnub.receive', ctx);

    try {
      process(e.message);
    } catch (err) {
      log.error('pubnub.receive.process_error', { ...ctx, error: err.message });
    }
  }
});
```

## Connection-State Logging

Connection events are the second-most-important thing to log:

```javascript
pubnub.addListener({
  status: (s) => {
    log.info('pubnub.status', {
      category: s.category,
      operation: s.operation,
      affectedChannels: s.affectedChannels,
      lastTimetoken: s.lastTimetoken,
      currentTimetoken: s.currentTimetoken
    });
  }
});
```

Categories like `PNNetworkDownCategory`, `PNReconnectedCategory`, `PNUnknownCategory` reveal connection health. See [pubnub-presence/references/dropped-connections.md](../../pubnub-presence/references/dropped-connections.md) for category meanings.

## Structured vs Free-Form

Always emit structured logs (JSON or key=value), never `console.log("Sent " + channel)`. Reasons:

- Searchable in a log aggregator (Datadog, ELK, Splunk, CloudWatch)
- Filterable by field
- Reliably parsed by alerting rules

```javascript
// Bad
console.log(`Sent message ${id} to ${channel}`);

// Good
log.info('pubnub.publish.ok', { channel, message_id: id, timetoken });
```

## Sampling High-Volume Streams

For high-publish-rate streams (telemetry, gaming), full logging is expensive. Sample by **`message_id` hash**, not random — so you keep all logs for a sampled message:

```javascript
function shouldSample(message_id, sampleRate = 0.01) {
  const h = simpleHash(message_id);
  return (h % 1000) < (sampleRate * 1000);
}

if (shouldSample(envelope.message_id, 0.01)) {
  log.info('pubnub.publish.sampled', ctx);
}
```

Always log **errors** and **status changes** in full (no sampling).

## Log Level Choice

| Event | Level |
|---|---|
| Successful publish (high volume) | `debug` or sampled `info` |
| Successful publish (low volume) | `info` |
| Successful receive | sampled `info` |
| Connection state change | `info` |
| Network down | `warn` |
| Publish error after retries | `error` |
| Receive process error | `error` |
| Reached `MAX_ATTEMPTS` in [reconnect](../../pubnub-reliability/references/backoff-and-jitter.md) | `error` (page on this) |

## Correlation Across Services

When a publisher's PubNub message triggers downstream work (a webhook, a Function, a server consumer), pass the `message_id` through. Every downstream log should carry it.

```javascript
// Webhook handler downstream of an Events & Actions integration
app.post('/webhook/pubnub', (req, res) => {
  const ctx = {
    channel: req.body.channel,
    message_id: req.body.message?.message_id,
    user_id: req.body.publisher,
    timetoken: req.body.timetoken
  };
  log.info('webhook.received', ctx);
  // ... process ...
  log.info('webhook.processed', ctx);
});
```

For [Events & Actions](../../pubnub-events-and-actions/references/event-types.md) integrations, this enables end-to-end tracing.

## What to Alert On

| Condition | Alert |
|---|---|
| `pubnub.publish.error` rate > X/min for Y minutes | Page on-call |
| `pubnub.receive.process_error` rate > X/min | Slack alert |
| `PNNetworkDownCategory` lasting > 60s on any client class | Slack alert |
| Reached `MAX_ATTEMPTS` reconnect | Page on-call |
| Sudden absence of `pubnub.receive` for a critical channel | Page on-call |

## Anti-Patterns

| Anti-pattern | Fix |
|---|---|
| Logging only the publish, not the response | Log both with the timetoken from the response |
| Free-form string logs | Structured key=value or JSON |
| Random sampling that drops correlated events | Sample by `message_id` hash |
| Forgetting to log the receiver's `user_id` | Add it; "what did this user see?" is a frequent question |
| Logging payload contents in chat | PII risk; log shape only or hash sensitive fields |
| No connection-state logging | Add it; status events explain 80% of "messages stopped" incidents |

## Related Reading

- [test-pyramid.md](test-pyramid.md) — what to test
- [incident-runbook.md](incident-runbook.md) — what to do when alerts fire
- [pubnub-reliability/references/idempotent-publish.md](../../pubnub-reliability/references/idempotent-publish.md) — `message_id` source
