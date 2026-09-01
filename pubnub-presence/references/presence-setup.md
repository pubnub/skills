<!-- canonical-for: PRESENCE_SETUP -->
<!-- used-by: -->

> **Cross-references:** Presence add-on enablement is part of [keyset configuration](../../pubnub-keyset-management/references/keysets-and-environments.md). For [SDK initialization](../../pubnub-app-developer/references/sdk-patterns.md) options including heartbeat and the [`pubnub.subscribe`](../../pubnub-app-developer/references/publish-subscribe.md) call shape see the canonical owners. Presence on restricted channels needs [Access Manager grants](../../pubnub-security/references/access-manager.md).

# Presence Setup — Decisions and Prerequisites


## Admin Portal prerequisites

1. Enable **Presence** add-on on the keyset.
2. Prefer **"Selected channels only"** — not all channels need occupancy tracking.
3. Configure **Presence Management channel rules** — **without rules, presence will not work** even if code calls `withPresence: true`.

## Heartbeat / timeout tuning

| Use case | Tuning direction |
|----------|------------------|
| Real-time chat / gaming | Shorter heartbeat & timeout — faster leave detection, more events |
| IoT / social / mobile battery | Longer intervals — fewer events, slower offline detection |

**Rule of thumb:** presence timeout ≥ **2×** heartbeat interval. Retrieve default and max values from SDK docs before hard-coding.

## Orchestration

1. Enable add-on + channel rules in Admin Portal.
2. Subscribe with presence on allowed channels only.
3. Seed UI with **`hereNow`** (or equivalent) for initial occupancy, then apply **join/leave/timeout** deltas from events — retrieve event types from SDK docs.
4. On high-occupancy channels, expect **interval** events — design UI to batch updates ([cost & payload hygiene](../../pubnub-observability/references/cost-and-payload-hygiene.md)).
5. Clean up subscriptions on unload so leave events fire promptly.

## Troubleshooting

| Symptom | Likely cause |
|---------|--------------|
| No presence events | Channel not listed in Presence Management rules |
| Occupancy always 0 | Add-on disabled or wrong keyset |
| Flapping online/offline | Timeout too aggressive vs mobile backgrounding — see [dropped-connections.md](dropped-connections.md) |
| Duplicate joins | Non-persistent `userId` — see [pubnub-app-developer/SKILL.md](../../pubnub-app-developer/SKILL.md) |

## Access Manager

When channels are token-gated, grant read access to both the data channel and its **`-pnpres`** companion — retrieve grant shape from Access Manager docs.
