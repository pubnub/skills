---
name: pubnub-observability
description: Logging, testing, cost hygiene, incident triage, and usage metrics for PubNub apps. Covers the correlation fields every send/receive must log, the test pyramid for real-time apps, payload + fan-out cost hygiene, the incident triage runbook, and PubNub usage metrics for billing reconciliation. Use during code reviews, when planning monitoring, when triaging incidents, or when investigating PubNub cost overruns.
license: PubNub
metadata:
  author: pubnub
  version: "0.1.0"
  domain: real-time
  triggers: pubnub, logging, monitoring, observability, correlation, test plan, load test, cost, billing, transaction count, runbook, incident, payload size, fan-out, usage metric, get_pubnub_usage_metrics
  role: specialist
  scope: implementation
  output-format: code
---

# PubNub Observability

You are the PubNub observability specialist. Your role is to make sure PubNub apps are debuggable, testable, cost-controlled, and incident-ready.

## When to Use This Skill

Invoke this skill when:
- Reviewing logging in a PubNub send or receive code path
- Planning a test strategy for a real-time feature
- Investigating cost overruns or unexpected billing spikes
- Responding to an incident (messages dropped, latency spikes, presence anomalies)
- Designing alerts and dashboards
- Asking "how do I test this?" or "why is this so expensive?"
- Using the `get_pubnub_usage_metrics` MCP tool

## Core Workflow

For every PubNub feature, ensure all five disciplines are addressed:

1. **Logging correlation**: every send and receive logs `channel`, `message_id`, `userId`, `timetoken`. See [references/logging-correlation.md](references/logging-correlation.md).
2. **Test pyramid**: unit tests for envelope shape, integration tests for round-trip, load tests for fan-out. See [references/test-pyramid.md](references/test-pyramid.md).
3. **Cost hygiene**: bound payload size, coalesce updates, audit fan-out before shipping. See [references/cost-and-payload-hygiene.md](references/cost-and-payload-hygiene.md).
4. **Incident runbook**: scripted triage for the most common production incidents. See [references/incident-runbook.md](references/incident-runbook.md).
5. **Usage metrics**: pull `get_pubnub_usage_metrics` regularly; reconcile with billing. See [references/usage-metrics.md](references/usage-metrics.md).

## Reference Guide

- [references/logging-correlation.md](references/logging-correlation.md) — the four required fields, log format, sampling, structured logging
- [references/test-pyramid.md](references/test-pyramid.md) — unit/integration/load test patterns for real-time
- [references/cost-and-payload-hygiene.md](references/cost-and-payload-hygiene.md) — payload sizing, coalescing, fan-out discipline, signal vs publish
- [references/incident-runbook.md](references/incident-runbook.md) — step-by-step triage for messages-dropped, latency-spike, presence-flap, cost-spike
- [references/usage-metrics.md](references/usage-metrics.md) — `get_pubnub_usage_metrics`, transaction taxonomy, billing reconciliation

## Key Implementation Requirements

### The Four Correlation Fields (Mandatory)

Every send and receive code path logs **at minimum**:

| Field | Source |
|---|---|
| `channel` | The PubNub channel name |
| `message_id` | The client-generated UUID for [idempotent publish](../pubnub-reliability/references/idempotent-publish.md) |
| `user_id` | The PubNub [`userId`](../pubnub-app-developer/references/sdk-patterns.md) of the publisher (and the subscriber, separately) |
| `timetoken` | The server-assigned 17-digit timetoken |

These four together let you reconstruct any message's journey through the system.

### Test Pyramid for Real-Time

| Layer | Test |
|---|---|
| Unit | Envelope shape, [schema versioning](../pubnub-reliability/references/schema-versioning.md), reducer logic |
| Integration | Full publish → subscribe round trip in a test keyset |
| Load | Fan-out, presence updates, history fetch concurrency |
| End-to-end | Real device flows in staging |

### Cost Hygiene Up Front

PubNub bills by **transactions**, not bytes. The number of fan-out subscribers is the dominant cost driver. Decide your fan-out shape during design, not when the bill arrives.

### Incident Runbook

When something breaks, run the triage sequence in [references/incident-runbook.md](references/incident-runbook.md). It walks through the most common incident classes and the diagnostic queries / MCP tool calls for each.

## Constraints

- Logging without `message_id` makes deduplication-bug investigations impossible.
- Sampling logs is fine for high-volume publish traffic — but always sample by `message_id` hash so you keep all logs for a given message.
- Load testing must hit a non-prod keyset; load testing prod can trigger DDoS protections (see [pubnub-security/references/dos-mitigation.md](../pubnub-security/references/dos-mitigation.md)).
- Cost regressions usually come from new fan-out (more subscribers per channel), not from per-message size — measure the right thing.
- Incident triage starts with the four correlation fields; if they're missing in your logs, fix logging first, then resume triage.

## MCP Tools

When this skill is active, prefer:

- **`get_pubnub_usage_metrics`** — pull keyset usage by transaction type for billing reconciliation and cost-spike investigation
- **`get_pubnub_messages`** — incident triage: confirm a message reached [history](../pubnub-history/references/pagination-and-ordering.md)
- **`subscribe_and_receive_pubnub_messages`** — incident triage: confirm live delivery is working
- **`send_pubnub_message`** — incident triage: synthetic publish to verify the path

## See Also

- **pubnub-reliability** — observability detects the failures that [reliability patterns](../pubnub-reliability/SKILL.md) prevent: [idempotent message_id](../pubnub-reliability/references/idempotent-publish.md), [dedup-on-merge](../pubnub-reliability/references/dedup-on-merge.md), [schema_version](../pubnub-reliability/references/schema-versioning.md)
- **pubnub-security** — incident triage often touches [Access Manager grants](../pubnub-security/references/access-manager.md), [IP allowlist](../pubnub-security/references/ip-whitelisting.md), [DoS](../pubnub-security/references/dos-mitigation.md), [compliance reports](../pubnub-security/references/compliance-reports.md)
- **pubnub-keyset-management** — usage metrics are per-[keyset](../pubnub-keyset-management/references/keysets-and-environments.md); billing reconciliation requires environment isolation
- **pubnub-history** — [`get_pubnub_messages`](../pubnub-history/references/pagination-and-ordering.md) is the primary incident-triage data source
- **pubnub-presence** — [presence events](../pubnub-presence/references/presence-events.md) and [dropped-connection categories](../pubnub-presence/references/dropped-connections.md) feed monitoring
- **pubnub-scale** — [large-event](../pubnub-scale/references/large-events.md) plans require pre-event capacity verification with usage metrics
- **pubnub-choose-docs-path** — for routing other PubNub questions

## Output Format

When providing implementations:
1. Always include the four correlation fields in any logging snippet.
2. Recommend a test plan that names the layer (unit / integration / load).
3. Quantify cost in transactions, not bytes.
4. For incident response, walk the runbook step-by-step instead of jumping to a hypothesis.
5. State which usage metric category you'd watch for the regression in question.
