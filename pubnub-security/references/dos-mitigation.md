<!-- canonical-for: DOS_MITIGATION -->
<!-- used-by: -->

> **Cross-references:** Layered with [Access Manager grants](access-manager.md), [IP allowlisting](ip-whitelisting.md), and [client-side backoff and jitter](../../pubnub-reliability/references/backoff-and-jitter.md). For incident-time triage see the [observability runbook](../../pubnub-observability/references/incident-runbook.md). Sustained traffic spikes show up in [usage metrics](../../pubnub-observability/references/usage-metrics.md). Identifies abusers by `userId` — see [SDK userId/UUID](../../pubnub-app-developer/references/sdk-patterns.md). The Function rate-limit pattern uses [Before Publish, `request.ok()`/`request.abort()`](../../pubnub-functions/references/functions-basics.md) and [`require('kvstore')`](../../pubnub-functions/references/functions-modules.md).

# DoS / DDoS Mitigation

PubNub edge POPs absorb large traffic volumes by design. Most denial-of-service patterns relevant to your app are **abuse from authenticated users** or **runaway clients**, not network-layer floods.

## Three Threat Models

### 1. Authenticated Abuse

A logged-in user publishes 10 messages per second to flood a chat room.

**Mitigations:**
- [Access Manager grants](access-manager.md) with short TTLs so revocation is fast.
- A [Before Publish Function](../../pubnub-functions/SKILL.md) that rate-limits per `userId` using KVStore counters.
- Block at the [App Context layer](../../pubnub-app-context/references/users.md) by setting a `banned: true` custom field that the Function checks.

### 2. Runaway Clients

A buggy client retries publish in a tight loop after every error.

**Mitigations:**
- Mandate [exponential backoff with jitter](../../pubnub-reliability/references/backoff-and-jitter.md) in client SDK setup.
- Bound the [retry queue](../../pubnub-reliability/references/queue-and-retry.md) so it can't grow forever.
- Watch for spikes in [transaction count](../../pubnub-observability/references/usage-metrics.md) per user.

### 3. Network-Layer Flood

Botnet aimed at PubNub edge.

**Mitigations:**
- PubNub absorbs and rate-limits at the edge — no action required from your code.
- Contact PubNub Support if you observe sustained edge errors.

## Hard Caps

The PubNub platform enforces hard caps per sub-key (publish rate, channel count, history retrieval rate). These are documented per plan tier in Admin Portal → Account → Plan & Limits. Build below them with headroom.

## Rate-Limit Function Pattern

```javascript
export default async (request) => {
  const db = require('kvstore');
  const userId = request.message.user_id;
  const key = `rl:${userId}`;
  const count = await db.get(key) || 0;
  if (count >= 10) {
    console.log(JSON.stringify({ blocked: true, user_id: userId, channel: request.channels[0] }));
    return request.abort();
  }
  await db.set(key, count + 1, 1);
  return request.ok();
};
```

This caps a single user at ~10 publishes per minute per pattern-matched channel. Tune for your app.

## Detection Checklist

- Sudden spike in [usage metrics](../../pubnub-observability/references/usage-metrics.md) without a corresponding user-count change.
- A handful of `userId`s contributing >50% of total traffic.
- Increase in `request.abort()` from a Before Publish Function.
- Surge in connection-state errors, suggesting upstream is throttling clients.

When detected, follow the [incident runbook](../../pubnub-observability/references/incident-runbook.md).

## Response Playbook

1. Identify the abusive `userId`(s) from logs (the [logging correlation owner](../../pubnub-observability/references/logging-correlation.md) requires `user_id`).
2. Revoke their tokens via [Access Manager](access-manager.md). Revocation can take up to 60s.
3. Set a `banned: true` flag in [App Context](../../pubnub-app-context/references/users.md) so the Function blocks immediately.
4. If volume continues, contact PubNub Support with sub-key + timeframe.
