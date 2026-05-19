<!-- canonical-for: LARGE_EVENTS -->
<!-- used-by: -->

> **Cross-references:** Builds on [scaling-patterns](scaling-patterns.md) (sharding, channel groups, wildcards) and [performance](performance.md) (signal vs publish, batching). Pair with [presence heartbeat tuning + presence event semantics](../../pubnub-presence/references/presence-events.md) (and [setup](../../pubnub-presence/references/presence-setup.md)), [reliability backoff and jitter](../../pubnub-reliability/references/backoff-and-jitter.md), and [usage-metrics monitoring](../../pubnub-observability/references/usage-metrics.md). For [DoS / abuse hardening during the event](../../pubnub-security/references/dos-mitigation.md) see the security owner. SDK setup uses [`pubnub.subscribe(` and listener wiring](../../pubnub-app-developer/references/publish-subscribe.md) plus [userId/UUID](../../pubnub-app-developer/references/sdk-patterns.md). Stage-tier fan-out commonly uses [Functions After Publish](../../pubnub-functions/references/functions-basics.md).

# Large-Event Playbook (10K+ Concurrent Users)

This page is the engagement model for live events: sports broadcasts, product launches, large-scale auctions, voting, anything that pushes a single sub-key into the tens of thousands of concurrent connections.

## Triggering Threshold

Engage PubNub Support **at least 2 weeks before** any of:
- 10,000+ concurrent subscribers on one sub-key
- 5,000+ concurrent subscribers on a single channel
- Sustained publish rate above your account's normal pattern by >5x
- Single message fan-out above 1M deliveries/min

These are not hard rejections — they are signals to pre-allocate capacity and reserve POPs.

## Pre-Event Checklist

### T-2 weeks
- [ ] Notify PubNub Support: sub-key, expected peak, expected duration, region(s).
- [ ] Confirm [Stream Controller](scaling-patterns.md) is enabled on your keyset.
- [ ] Validate your channel-sharding strategy with realistic load test.
- [ ] Confirm your client SDKs are on a supported version.

### T-1 week
- [ ] Run a load test at 50% of expected peak.
- [ ] Tune client [heartbeat to 60-120s](../../pubnub-presence/references/presence-setup.md) (lowers presence cost).
- [ ] Ensure clients use [exponential backoff with jitter](../../pubnub-reliability/references/backoff-and-jitter.md) on reconnect.
- [ ] Set up live dashboards on [usage metrics](../../pubnub-observability/references/usage-metrics.md).
- [ ] Pre-warm [App Context user metadata](../../pubnub-app-context/references/users.md) for known users.

### T-1 day
- [ ] Brief on-call.
- [ ] Confirm [incident runbook](../../pubnub-observability/references/incident-runbook.md) is up-to-date.
- [ ] Verify [Access Manager grants](../../pubnub-security/references/access-manager.md) are issued with sensible TTLs (event duration + buffer).

### T-0 (event live)
- [ ] Watch transaction count, error rate, presence event volume in real time.
- [ ] Be ready to revoke abusive `userId`s via [Access Manager](../../pubnub-security/references/access-manager.md).
- [ ] If shed-load is needed, fall back to [signal vs publish](performance.md) for non-critical events.

### T+1 day
- [ ] Review [usage metrics](../../pubnub-observability/references/usage-metrics.md) for cost spikes.
- [ ] Capture lessons; update this page.

## Channel Sharding Pattern

For a million-viewer broadcast, do **not** put every viewer on `event.live`. Shard:

```javascript
const SHARD_COUNT = 100;
const userId = 'user-12345';
const shardId = hashToShard(userId, SHARD_COUNT);
const channel = `event.live.shard${shardId}`;

pubnub.subscribe({ channels: [channel] });
```

Then publish the same payload to all shards (server-side fan-out, or via [Function chaining](../../pubnub-functions/references/functions-chaining.md), or via parallel publishes).

## Stage Hierarchy

For very large events, use a **stage** model:
- **Origin channel** — your CMS or game server publishes once.
- **Fan-out tier** — a [Function](../../pubnub-functions/SKILL.md) (After Publish) re-publishes to N shard channels.
- **Subscriber channels** — clients subscribe to one shard.

This isolates publisher rate from subscriber count.

## Common Pitfalls

| Pitfall | Mitigation |
|---|---|
| All clients reconnect at once after a transient blip | Mandatory [backoff + jitter](../../pubnub-reliability/references/backoff-and-jitter.md) |
| Presence storm at event start | Stagger client connect over 60-120s; tune [heartbeat interval](../../pubnub-presence/references/presence-setup.md) |
| Single-channel hot spot | Shard channels |
| Token refresh storm | Issue tokens with TTL > event duration |
| Cost spike post-event | Pre-set alerts on [usage metrics](../../pubnub-observability/references/usage-metrics.md) |
| Pause-the-world publishes from the publisher app | Use signal for non-critical telemetry; see [performance.md](performance.md) |

## After-Action

For every large event, file an after-action covering:
1. Peak concurrent subscribers (per channel, per shard, total).
2. Peak publish rate.
3. Total messages, total transactions.
4. Cost vs forecast.
5. Anything from the [incident runbook](../../pubnub-observability/references/incident-runbook.md) that fired.
6. Things to do differently next time — feed back into this page.
