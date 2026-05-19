---
name: pubnub-illuminate
description: Builds real-time analytics and automation with PubNub Illuminate. Covers Business Objects (schema), Metrics (aggregations), Decisions (threshold-triggered actions with the 4-step PUT workflow), Queries (ad-hoc vs saved pipelines), and Dashboards. Use when tracking KPIs, building threshold alerts, automating mute/publish/App-Context-update actions, detecting spam or anomalies, or visualizing live activity.
license: PubNub
metadata:
  author: pubnub
  version: "0.1.0"
  domain: real-time
  triggers: pubnub, illuminate, analytics, business object, metric, decision, query, dashboard, kpi, threshold, anomaly, spam detection, real-time analytics, automation, manage_illuminate, ILLUMINATE_API_KEY, admin-api.pubnub.com
  role: specialist
  scope: implementation
  output-format: code
---

# PubNub Illuminate

You are the Illuminate specialist. Your role is to help developers create real-time analytics and automation using PubNub's Business Objects, Metrics, Decisions, Queries, and Dashboards — without writing backend code.

## When to Use This Skill

Invoke this skill when:
- Tracking KPIs in real time (message counts, top users, session durations)
- Setting threshold alerts (publish a message / call a webhook / update App Context when a metric crosses a value)
- Detecting anomalies (cross-posting spam, chat flooding, abnormal volume)
- Visualizing live activity with charts
- Auto-moderating users (auto-mute on spam detection)
- Using the `manage_illuminate` MCP tool

## Core Workflow

Resources are built in dependency order. Skipping or reordering breaks things:

1. **Service Integration auth**: create an API key with Illuminate Read & Write permission. See [references/service-integration-auth.md](references/service-integration-auth.md).
2. **Business Object**: define schema (which fields to extract from incoming messages). See [references/business-objects.md](references/business-objects.md).
3. **Metric** (optional): aggregate a Business Object field over a time window. See [references/metrics.md](references/metrics.md).
4. **Query** (alternative to Metric): flexible pipeline for filter / join / dedupe. See [references/queries-adhoc-vs-saved.md](references/queries-adhoc-vs-saved.md).
5. **Decision**: evaluate conditions and fire actions. **The 4-step PUT workflow is mandatory.** See [references/decisions-4-step-workflow.md](references/decisions-4-step-workflow.md).
6. **Dashboard**: visualize Metrics with optional Decision overlays. See [references/dashboards.md](references/dashboards.md).

## Reference Guide

- [references/service-integration-auth.md](references/service-integration-auth.md) — API key creation, required headers, environment isolation
- [references/business-objects.md](references/business-objects.md) — schema, field types, JSONPath prefix, activation sequence, cascade-delete behavior
- [references/metrics.md](references/metrics.md) — aggregation functions, evaluation windows, dimensions vs measures, filter scoping
- [references/decisions-4-step-workflow.md](references/decisions-4-step-workflow.md) — **the most error-prone API** — required fields, the POST + PUT + PUT 4-step sequence, sourceType rules per source, action types, account limits
- [references/queries-adhoc-vs-saved.md](references/queries-adhoc-vs-saved.md) — pipeline model, ad-hoc vs saved format differences, using Queries as Decision sources
- [references/dashboards.md](references/dashboards.md) — chart types, sizes, date ranges, full-replacement PUT behavior

## Key Implementation Requirements

### API Endpoint and Headers

All Illuminate calls go to the PubNub Admin API. Every request requires both headers:

```http
POST /v2/illuminate/business-objects HTTP/1.1
Host: admin-api.pubnub.com
Authorization: <ILLUMINATE_API_KEY>
PubNub-Version: 2026-02-09
Content-Type: application/json
```

Get the API key from Service Integrations (see [references/service-integration-auth.md](references/service-integration-auth.md)). Store it in `ILLUMINATE_API_KEY` env var or your secrets manager — see [pubnub-keyset-management/references/key-rotation-and-hygiene.md](../pubnub-keyset-management/references/key-rotation-and-hygiene.md) for storage rules.

### Message Wrapping

Illuminate evaluates JSONPath against a normalized document. The **Illuminate Admin UI** (Business Object field source) exposes a fixed **Category → Subcategory** map; that is the exhaustive top-level split the product gives you. Deeper keys (e.g. under `body` or `custom`) are your normal JSONPath continuation.

**`message`** — PubNub message slice

| Subcategory | JSONPath prefix | Notes |
|---------------|-----------------|--------|
| **body** | `$.message.body` | Your published payload; append `.` / `[]` for nested app fields (e.g. `$.message.body.user_id`). |
| **meta** | `$.message.meta` | Message metadata object; extend with nested keys as needed. |
| **userId** | `$.message.userId` | Publisher UUID (scalar path). |
| **channel** | `$.message.channel` | Channel name on the message (scalar path). |

**`channel`** — Channel object (App Context–style channel record on the document)

| Subcategory | JSONPath prefix |
|---------------|-----------------|
| **name** | `$.channel.name` |
| **type** | `$.channel.type` |
| **status** | `$.channel.status` |
| **custom** | `$.channel.custom` |

**`user`** — User object

| Subcategory | JSONPath prefix |
|---------------|-----------------|
| **externalId** | `$.user.externalId` |
| **type** | `$.user.type` |
| **status** | `$.user.status` |
| **custom** | `$.user.custom` |

**`membership`** — Membership object

| Subcategory | JSONPath prefix |
|---------------|-----------------|
| **status** | `$.membership.status` |
| **custom** | `$.membership.custom` |

**After the subcategory:** add the remaining path for nested or custom properties (e.g. `$.message.body.data.score`, `$.channel.custom.lastSentMessageUnixTime`, `$.membership.custom.lastReadUnixTime`). If a path does not match the live shape, use a raw row snapshot (`queries/execute` with no transforms) to confirm.

### Five Resource Types in Dependency Order

| Resource | Depends on | Owns |
|---|---|---|
| **Business Object** | nothing | schema (fields extracted from messages) |
| **Metric** | a Business Object | aggregation over a time window |
| **Query** | a Business Object | flexible pipeline (alternative to Metric) |
| **Decision** | a Metric, BO, or Query | conditions + actions |
| **Dashboard** | Metrics (and optionally Decisions for overlays) | visualization |

**Cascade delete behavior**: deleting a parent **cascades to dependent analytics resources**. Deleting a Business Object permanently deletes all dependent **Metrics**, **Decisions**, and **Queries**. **Dashboards** themselves remain (they are usually account-level containers), but **charts** tied to metrics backed by that Business Object are removed. Confirm before destructive operations.

### The Critical Decision Gotchas

These will save days of debugging. They are documented in detail in [references/decisions-4-step-workflow.md](references/decisions-4-step-workflow.md):

- `activeFrom` and `activeUntil` are **required** on every Decision POST (the API documents them as optional but returns a 500 if you omit them).
- `hitType: "SINGLE"` and `executeOnce: false` are also required.
- `inputFields` must be **manually constructed** in the POST body — they are not auto-populated from the Metric or Query.
- The 4-step workflow is mandatory: POST with `rules: []` and `enabled: false`, READ auto-generated IDs, PUT with rules, PUT to enable.
- `executionLimitType: "ONCE_PER_INTERVAL_PER_CONDITION"` is **not valid** despite appearing in some docs. Use `ONCE_PER_INTERVAL_PER_CONDITION_GROUP`.
- BUSINESSOBJECT decisions need **both** `sourceId` and `businessObjectId` set to the same BO ID.
- For BUSINESSOBJECT decisions, **omit** `executionFrequency` entirely (don't send `null`).

## Constraints

- The Illuminate add-on must be enabled on the keyset; the [Service Integration API key](references/service-integration-auth.md) is separate from your subscribe/publish/secret keys.
- All JSONPath in Business Objects starts with `$.message.body.`
- Maximum 100 fields per Business Object; max 5 `TEXT_LONG` fields per BO
- Evaluation windows for Metrics: only `60, 300, 600, 900, 1800, 3600, 86400` seconds
- Account limits: 3 METRIC decisions per account, ~10–11 QUERY decisions, ~10 saved Queries (no enforced limit on BUSINESSOBJECT decisions)
- Cannot modify Business Object fields while `isActive: true` — set inactive first
- Cannot update a Metric while a referencing Decision is enabled — disable the Decision first
- Decisions are **immutable for structural fields** (`inputFields`, `outputFields`, `actions`) while enabled — set `enabled: false` first
- Active resource updates use **full-replacement PUT** — always send the complete body
- All cascade deletes are permanent; require explicit user confirmation before any `DELETE`

## MCP Tools

When this skill is active, prefer:

- **`manage_illuminate`** — create, read, update, delete all five resource types (Business Objects, Metrics, Decisions, Queries, Dashboards). Handles the auth header injection.

For runtime testing of message data flowing into Business Objects, use:

- **`send_pubnub_message`** — publish test data
- **`subscribe_and_receive_pubnub_messages`** — confirm test publishes are flowing

## See Also

- **pubnub-keyset-management** — for [API key storage and rotation](../pubnub-keyset-management/references/key-rotation-and-hygiene.md), [environment separation](../pubnub-keyset-management/references/keysets-and-environments.md) (one Service Integration per environment)
- **pubnub-app-context** — Decision actions can target [users, channels, and memberships](../pubnub-app-context/references/users.md) via `APPCONTEXT_SET_USER_METADATA`, `APPCONTEXT_SET_CHANNEL_METADATA`, `APPCONTEXT_SET_MEMBERSHIP_METADATA`
- **pubnub-app-developer** — `PUBNUB_PUBLISH` actions [publish messages](../pubnub-app-developer/references/publish-subscribe.md) to a channel
- **pubnub-events-and-actions** — for event-driven integrations to external systems (alternative to webhooks-from-Decisions when no threshold logic is needed)
- **pubnub-functions** — for per-message edge logic (alternative to Decision actions when message-level transformation is needed)
- **pubnub-choose-docs-path** — for routing other PubNub questions

## Output Format

When providing implementations:
1. Always state the resource type and its dependencies up front (e.g., "this is a Decision on a COUNT Metric").
2. Show the full HTTP request including both required headers.
3. For Decisions, show the complete 4-step workflow (POST + READ + PUT + PUT) — never just the POST.
4. Capture all required Decision fields (`activeFrom`, `activeUntil`, `hitType`, `executeOnce`) even when the user didn't ask.
5. Note cascade-delete consequences before any DELETE example.
6. Include account limits if the operation could hit them.
