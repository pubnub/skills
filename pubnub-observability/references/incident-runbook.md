<!-- canonical-for: INCIDENT_RUNBOOK -->
<!-- used-by: pubnub-keyset-management, pubnub-history, pubnub-reliability, pubnub-security, pubnub-choose-docs-path -->

# Incident Runbook

The canonical reference for triaging the most common PubNub-related production incidents.

## General Triage Order

For **any** PubNub incident, run this sequence first:

1. **Identify the four [correlation fields](logging-correlation.md)** for the affected message(s) or user(s): `channel`, `message_id`, `user_id`, `timetoken`.
2. **Check PubNub status**: open `https://status.pubnub.com` and verify no platform-wide incident.
3. **Check `get_pubnub_usage_metrics`** for the affected keyset and time window — see [usage-metrics.md](usage-metrics.md). A traffic anomaly often correlates with the incident.
4. **Run the runbook** for the specific incident class.

## Runbook 1: "Messages Are Being Dropped"

### Symptoms

- Subscribers report missing messages
- Receive logs are missing `message_id`s that publish logs show were sent

### Triage

1. **Verify the publish actually reached PubNub.** Confirm the publisher's `pubnub.publish.ok` log includes a `timetoken`. If only `pubnub.publish.start` is present, the publisher never got an ack.

2. **Confirm the message landed in [history](../../pubnub-history/references/pagination-and-ordering.md)** (assuming Message Persistence is on):
   ```bash
   # Use get_pubnub_messages MCP tool with channel + timetoken bounds
   ```
   If the message is in history → publish succeeded; receive side is the bug.
   If the message is NOT in history → publish path has the bug or the message was sent with `storeInHistory: false`.

3. **Check receive-side dedup.** The receiver may be dropping the message as a "duplicate" because [`message_id`](../../pubnub-reliability/references/idempotent-publish.md) collided. Inspect the [dedup set](../../pubnub-reliability/references/dedup-on-merge.md).

4. **Check Access Manager.** If the subscriber's [token](../../pubnub-security/references/access-manager.md) doesn't grant read on the channel, messages are silently filtered. Look for `PNAccessDeniedCategory` in status logs.

5. **Check connection state.** If subscriber was in `PNNetworkDownCategory` at the publish moment and missed the [short-term cache window](../../pubnub-history/references/offline-catch-up.md), the message is gone unless explicitly fetched from history.

### Resolution

- If publish-side: investigate publisher network, retry policy, [queue](../../pubnub-reliability/references/queue-and-retry.md) drain.
- If receive-side: fix dedup logic or grant logic.
- If subscriber-was-disconnected: implement [offline catch-up](../../pubnub-history/references/offline-catch-up.md) using stored history.

## Runbook 2: "Latency Spike — Messages Are Slow"

### Symptoms

- p99 receive latency > 500ms (was < 100ms)
- Users report "feels laggy"

### Triage

1. **Check `get_pubnub_usage_metrics`**: did publish rate or fan-out spike? See [cost-and-payload-hygiene.md](cost-and-payload-hygiene.md).

2. **Check payload sizes.** Did the average payload grow? A 10x payload size increase 10x the serialization and bandwidth cost.

3. **Check for new subscribers.** Did a new client cohort start subscribing to the channel, increasing fan-out?

4. **Check PubNub status page** for regional incidents.

5. **Check for retry storms.** If many clients are reconnecting in lockstep (no [jitter](../../pubnub-reliability/references/backoff-and-jitter.md)), that creates a thundering herd.

6. **Check Function execution time** if a `before publish` Function is in the path. A slow Function blocks the publish. See [pubnub-functions/references/functions-basics.md](../../pubnub-functions/references/functions-basics.md).

### Resolution

- Spike in publish rate: coalesce publishes (see [cost-and-payload-hygiene.md](cost-and-payload-hygiene.md)).
- Larger payloads: trim, move large data to [File Sharing](../../pubnub-chat/references/file-sharing.md).
- Slow Function: optimize or move work async.
- Retry storm: enforce jitter; cap reconnect attempts.

## Runbook 3: "Cost Spike"

### Symptoms

- Bill is N× last month
- `get_pubnub_usage_metrics` shows a transaction-count spike

### Triage

1. **Pull [usage metrics](usage-metrics.md)** for the spike window. Categorize by transaction type (publish, subscribe, function, history, app context, presence).

2. **Identify the dominant transaction type.** Most cost spikes come from one category.

3. **For publish/subscribe spikes**: check if a new feature shipped that increased fan-out or publish rate. Cross-reference with deploy timestamps.

4. **For function spikes**: check if a Function is firing on every event when it should be filtered.

5. **For history spikes**: check if a client started polling history instead of subscribing live.

6. **For app context spikes**: check if a client is reading user metadata on every render instead of caching.

### Resolution

- Apply [coalescing](cost-and-payload-hygiene.md) where appropriate.
- Add [Function](../../pubnub-functions/references/functions-basics.md) filters before expensive work.
- Cache [app context user metadata](../../pubnub-app-context/references/users.md) locally; subscribe to [metadata events](../../pubnub-app-context/references/metadata-and-filtering.md) for invalidation.
- Migrate history-polling to live subscription where possible.

## Runbook 4: "Presence Is Wrong / Flapping"

### Symptoms

- `hereNow` returns the wrong member count
- Users see "user joined" / "user left" rapid-fire

### Triage

1. **Check heartbeat configuration.** Default is 300s heartbeat; if this is too long, slow joins/leaves; too short, false flapping. See [pubnub-presence/references/presence-events.md](../../pubnub-presence/references/presence-events.md).

2. **Check for duplicate [`userId`](../../pubnub-app-developer/references/sdk-patterns.md)s.** Two devices using the same `userId` (e.g., a logged-in user with two browser tabs) interact with presence in subtle ways. See [pubnub-presence/references/multi-device-sync.md](../../pubnub-presence/references/multi-device-sync.md).

3. **Check `PNNetworkDownCategory` rate.** If many clients are flapping connections, presence will flap. See [pubnub-presence/references/dropped-connections.md](../../pubnub-presence/references/dropped-connections.md).

4. **Check Presence add-on configuration.** Verify it's enabled on the keyset.

### Resolution

- Tune heartbeat (180s common middle-ground).
- Implement application-level "online" with debouncing if flapping is unavoidable.
- Use a different `userId` per device or accept that presence is per-connection, not per-user.

## Runbook 5: "Subscribe Stops Receiving Suddenly"

### Symptoms

- Subscriber was working, suddenly stops
- No errors logged

### Triage

1. **Check `pubnub.status` logs.** Most "stopped receiving" incidents are an unhandled status event. Look for `PNAccessDeniedCategory`, `PNNetworkDownCategory`, `PNTimeoutCategory`.

2. **Check Access Manager token expiration.** If the [token TTL](../../pubnub-security/references/access-manager.md) elapsed, the subscriber loses permission silently. Server should re-issue.

3. **Check for keyset-level issues:** [DDoS mitigation](../../pubnub-security/references/dos-mitigation.md) may have throttled the client; [IP allowlist](../../pubnub-security/references/ip-whitelisting.md) may have changed.

4. **Verify the publish side is still publishing.** The "subscribe stopped" might actually be "publish stopped".

### Resolution

- Implement automatic [token refresh](../../pubnub-security/references/access-manager.md).
- Handle status events explicitly; never silently swallow.
- For DoS-flagged clients: contact PubNub Support with the userId and time window.

## Runbook 6: "Webhook / Events & Actions Integration Stopped"

### Symptoms

- Downstream integration (Lambda, Kafka, webhook) no longer receives events

### Triage

1. **Check Events & Actions delivery logs in the Admin Portal.** See [pubnub-events-and-actions/references/event-types.md](../../pubnub-events-and-actions/references/event-types.md).

2. **Check the integration's auth.** Did the webhook URL change? Did the Lambda role rotate? Did Kafka credentials expire?

3. **Check [keyset](../../pubnub-keyset-management/references/keysets-and-environments.md) the integration is bound to.** Sometimes integrations get bound to the wrong env's keyset.

4. **Check for retries.** Events & Actions retries failures; long-lived failures eventually move to dead letter.

### Resolution

- Refresh integration credentials.
- Re-create the integration if it was disabled by repeated failures.

## Runbook 7: "Illuminate Decision Fired Wrong (or Didn't Fire)"

### Symptoms

- Threshold was crossed but no action fired
- Action fired when it shouldn't have

### Triage

1. **Check the Decision is `enabled: true`.** See [pubnub-illuminate/references/decisions-4-step-workflow.md](../../pubnub-illuminate/references/decisions-4-step-workflow.md).

2. **Verify input field bindings.** Did the underlying [Metric](../../pubnub-illuminate/references/metrics.md) or [Business Object](../../pubnub-illuminate/references/business-objects.md) get updated, breaking field IDs?

3. **Check rule conditions.** A `NUMERIC_GREATER_THAN` of 100 doesn't fire on 100; use `NUMERIC_GREATER_THAN_EQUALS` if needed.

4. **Check `executionLimitType`.** `ONCE` and `ONCE_PER_INTERVAL_PER_CONDITION_GROUP` suppress repeat firings.

5. **Verify message envelope shape.** A [schema_version bump](../../pubnub-reliability/references/schema-versioning.md) might have broken the BO's JSONPath.

### Resolution

- Fix Decision configuration per workflow rules.
- Re-version BO and Decision after schema changes.

## Runbook 8: "We Pushed and Now Everything Is Broken"

### Symptoms

- Sudden incident immediately after a deploy

### Triage

1. **Roll back the deploy** if confidence is low. PubNub-side state is mostly reversible.

2. **Check for [SDK upgrades](../../pubnub-app-developer/references/sdk-upgrades.md).** A breaking SDK change is a common silent failure mode.

3. **Check for [schema_version](../../pubnub-reliability/references/schema-versioning.md) bumps.** Did the producer ship before receivers were updated to handle the new version?

4. **Check [`userId`](../../pubnub-app-developer/references/sdk-patterns.md) changes.** If `userId` format changed (e.g., started using emails instead of UUIDs), App Context, Access Manager, and presence all get confused.

5. **Check Access Manager token format.** Did the new code start passing `token` instead of `authKey`? See [pubnub-security/references/access-manager.md](../../pubnub-security/references/access-manager.md).

### Resolution

- Roll back. Investigate from a stable baseline.
- Re-deploy with proper version coordination (receivers first).

## Communication Template

For any production incident, post to the war-room channel with:

```
INCIDENT: <one-line summary>
Started: <timestamp>
Affected: <users / region / feature>
Symptoms: <what users see>
Hypothesis: <current best guess>
PubNub status page: <green / yellow / red>
Usage metrics anomaly: <yes/no, what>
Active correlation fields: channel=X, message_ids=[...]
Runbook in progress: <runbook name>
On-call lead: <name>
```

Update on every state change.

## Related Reading

- [logging-correlation.md](logging-correlation.md) — the four fields you need for any triage
- [usage-metrics.md](usage-metrics.md) — first stop for cost-related incidents
- [pubnub-reliability/SKILL.md](../../pubnub-reliability/SKILL.md) — patterns that prevent these incidents
