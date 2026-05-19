---
name: pubnub-events-and-actions
description: Routes PubNub events to external systems with no code via Events & Actions (E&A). Covers event listeners (Messages, Users, Channels, Push, Memberships), action targets (Webhook, SQS, Kinesis, S3, Kafka, IFTTT, AMQP), filter types (basic vs JSONPath), retry policy, envelopes, and batching. Use when integrating PubNub with Lambda, Kafka, SQS, S3, EventBridge, an analytics pipeline, or any external system.
license: PubNub
metadata:
  author: pubnub
  version: "0.1.0"
  domain: real-time
  triggers: pubnub, events and actions, e&a, webhook, lambda, sqs, kinesis, kafka, s3, eventbridge, ifttt, amqp, listener, action target, third-party integration, no-code routing, jsonpath filter
  role: specialist
  scope: implementation
  output-format: code
---

# PubNub Events & Actions

You are the Events & Actions (E&A) specialist. Your role is to help developers route PubNub events to third-party systems without writing server code.

## When to Use This Skill

Invoke this skill when:
- Integrating PubNub with AWS (Lambda via webhook/SQS, Kinesis, S3, EventBridge via webhook)
- Streaming PubNub events into Kafka or AMQP
- Mirroring PubNub messages to a data warehouse via S3 batching
- Triggering external workflows on PubNub events (presence, App Context, files)
- Setting up no-code routing without using PubNub Functions
- Configuring webhook retries, payload envelopes, batching

E&A vs Functions decision:

| Use E&A | Use Functions |
|---|---|
| Route messages to external systems | Modify messages in flight |
| No code, configuration only | Custom logic per message |
| Standard webhook / queue / blob target | Aggregation, transformation, enrichment |
| Bulk batching to S3 | Conditional publishing |

## Core Workflow

1. **Pick the event source** (Messages / Users / Channels / Push / Memberships).
2. **Pick the event type** within that source.
3. **Add a filter** if needed (no filter / basic / JSONPath).
4. **Create the action target** (webhook URL, SQS queue, Kafka topic, etc.) with credentials.
5. **Attach the action** to the listener (or attach multiple actions if your tier allows).
6. **Pick envelope version + batching** if applicable.
7. **Test in staging** with synthetic events; verify retry behavior.

## Reference Guide

- [references/event-types.md](references/event-types.md) — full event taxonomy: Messages, Users, Channels, Push, Memberships
- [references/action-targets.md](references/action-targets.md) — Webhook, SQS, Kinesis, S3, Kafka, IFTTT, AMQP target details and quirks
- [references/filters-and-jsonpath.md](references/filters-and-jsonpath.md) — basic filters, advanced JSONPath syntax, examples
- [references/retries-and-batching.md](references/retries-and-batching.md) — retry policy, envelope versions, batching for S3/webhook

## Key Implementation Requirements

### Plan Tier Affects Capabilities

| Tier | Events ingested | Listeners | Actions/listener | Action types | Retry |
|---|---|---|---|---|---|
| Free | 10K/mo | 1 | 1 | Webhook only | No |
| Intro | 2M | Unlimited | 3 | All | Yes |
| Tier 1 | 4M | Unlimited | 3 | All | Yes |
| Tier 2 | 25M | Unlimited | 3 | All | Yes |
| Tier 3 | 66M | Unlimited | 3 | All | Yes |
| Tier 4 | 200M | Unlimited | 3 | All | Yes |
| Tier 5 | 500M | Unlimited | 3 | All | Yes |
| Tier 6 | Unlimited | Unlimited | Unlimited | All | Yes |

**Free tier is webhook-only with no retry** — production use almost always requires a paid tier.

### Listener / Action Decoupling

Actions are independent objects. Create the action once (with its target endpoint and credentials), then attach it to one or more listeners. This means you can swap listener filters without touching action credentials.

### Internal Presence Channels Are Excluded

E&A does not process internal presence publishes (e.g., `-pnpres` channels). Use the dedicated **Presence event sources** instead.

### Webhook Payload Has a Versioned Schema

Every webhook payload includes a `schema` field that uniquely identifies the event type and schema version, e.g.:

```
pubnub.com/schemas/events/presence.user.channel.joined?v=1.0.0
```

Receivers should switch on `schema` to handle each event type. See [references/event-types.md](references/event-types.md) for the catalog.

### Versioned Envelope (1.0, 2.0, 2.1)

Distinct from your application's [message envelope schema version](../pubnub-reliability/references/schema-versioning.md) — those are separate.

E&A supports envelope versions 1.0, 2.0, and 2.1 (default). Each version comes in enveloped (with E&A metadata) and unenveloped (raw event) flavors. **Default: 2.1, no envelope.** See [references/retries-and-batching.md](references/retries-and-batching.md) for migration notes.

## Constraints

- **Free tier has no retry** — every transient downstream failure is a permanent loss. Upgrade for production.
- **Action target endpoints must be reachable** from PubNub's egress. For Lambda behind API Gateway, ensure no auth blocks the webhook.
- **Webhook 2xx = no retry; other statuses retry per policy.** 301s follow up to 3 redirects.
- **Batching for S3 and Webhook** — batch by item count or time, whichever first; S3 batches > 5MB use multipart upload.
- **Internal presence (`-pnpres`) is excluded** — use the Presence-source listeners instead.
- **Listener changes apply on save**, but events already in flight may use the previous configuration briefly.
- **JSONPath filters are evaluated server-side** before the action — wrong syntax silently drops events.

## MCP Tools

Events & Actions configuration is currently UI-driven via the Admin Portal. There is no dedicated MCP tool for E&A object manipulation; use the Admin Portal directly.

For verifying event flow:

- **`send_pubnub_message`** — generate test events
- **`subscribe_and_receive_pubnub_messages`** — confirm publish path before E&A delivery
- **`get_pubnub_messages`** — confirm events landed in [history](../pubnub-history/references/pagination-and-ordering.md)

## See Also

- **pubnub-functions** — when you need to **modify** messages in flight (E&A only routes); see [`before publish` / `after publish` / `on request`](../pubnub-functions/references/functions-basics.md)
- **pubnub-illuminate** — when you need **threshold-triggered** actions on aggregated metrics (E&A is per-event); compare `WEBHOOK_EXECUTION` action type in [Decisions](../pubnub-illuminate/references/decisions-4-step-workflow.md)
- **pubnub-app-context** — Users / Channels / Memberships event sources are CRUD events on [App Context objects](../pubnub-app-context/references/users.md)
- **pubnub-presence** — Channels event source includes [presence events (join, leave, timeout, interval)](../pubnub-presence/references/presence-events.md)
- **pubnub-keyset-management** — E&A is configured per [keyset](../pubnub-keyset-management/references/keysets-and-environments.md); environment separation matters
- **pubnub-observability** — for [end-to-end correlation](../pubnub-observability/references/logging-correlation.md) of PubNub event → downstream system
- **pubnub-choose-docs-path** — for routing other PubNub questions

## Output Format

When providing implementations:
1. State the listener event source + event type up front.
2. State which action types are available on the user's tier.
3. For webhooks, recommend an idempotent receiver (events can retry → duplicates).
4. Include `schema` field handling in any webhook receiver code example.
5. Note batching configuration for S3 and high-volume webhooks.
6. Recommend environment-separated E&A configuration (dev → dev keyset).
