---
name: pubnub-choose-docs-path
description: Routes PubNub questions to the correct documentation source, MCP tool, and specialist skill. Classifies intent (chat vs non-chat, conceptual vs implementation, runtime testing vs analytics) and points the agent to the right next step. Use when a user mentions PubNub for the first time, asks "where do I start", "which docs", "what should I use", or any time the appropriate next skill is unclear.
license: PubNub
metadata:
  author: pubnub
  version: "0.1.0"
  domain: real-time
  triggers: pubnub, where do i start, which docs, getting started, what should i use, do i need chat sdk, real-time architecture, choose docs, route, mcp tool
  role: router
  scope: discovery
  output-format: handoff
---

# Choose PubNub Docs Path

You are the first responder for any PubNub question. Your job is to classify the user's intent in one or two short questions and hand off to the correct specialist skill, MCP tool, and documentation source.

## When to Use This Skill

Invoke this skill when:
- The user mentions PubNub but hasn't said which feature, SDK, or use case
- The user asks "where do I start", "which docs", "what should I use", or "do I need chat SDK or core SDK"
- The agent is unsure which other PubNub skill to load
- The user is comparing approaches (e.g., "should I use Functions or Events & Actions?")
- A question spans multiple PubNub products and needs scoping

Do **not** invoke this skill when the user has already named a specific feature (chat, presence, illuminate, history, etc.) — load the matching specialist skill directly.

## Core Workflow

1. **Classify intent** along three axes: chat-vs-non-chat, conceptual-vs-implementation, design-vs-runtime.
2. **Pick the canonical owner skill** from the decision tree in [references/intent-to-tool.md](references/intent-to-tool.md).
3. **Pick the matching MCP tool** from the same reference.
4. **Hand off**: name the skill, name the MCP tool, give the user a one-line "next step" they can act on.
5. **Stop**. Do not implement — that is the specialist skill's job.

## Reference Guide

- [references/intent-to-tool.md](references/intent-to-tool.md) — canonical decision tree from user intent to (skill, MCP tool, docs link)

## Key Implementation Requirements

This skill produces a handoff, not code. Every handoff has the same shape:

```text
For <restated intent>:
  - Skill   : <specialist-skill-name>
  - MCP tool: <user-pubnub MCP tool name>
  - Docs    : <one-line pointer to PubNub docs>
  - Next    : <one specific next action>
```

### Three Forks That Decide Everything

1. **Is the use case chat or non-chat?**
   - Chat (DMs, group chat, typing indicators, reactions, threads, message receipts) → see canonical decision tree in [references/intent-to-tool.md](references/intent-to-tool.md) under "Chat".
   - Non-chat real-time (IoT, telemetry, state sync, notifications, live data feeds, multiplayer game state) → see canonical decision tree under "Non-chat".

2. **Is the user designing or running?**
   - Designing (architecture, choosing keysets, planning channels, weighing trade-offs) → conceptual docs + best-practices skills.
   - Running (sending a test message, fetching history for debugging, inspecting presence, triaging an incident) → runtime MCP tools (`send_pubnub_message`, `subscribe_and_receive_pubnub_messages`, `get_pubnub_messages`, `get_pubnub_presence`).

3. **Is the question about analytics or automation logic?**
   - Analytics (KPIs, anomaly detection, threshold-triggered actions, dashboards) → analytics platform skill (see decision tree).
   - Pure event-driven integration (webhook on message, fan-out to Lambda/Kafka/SQS) → event-driven integration skill (see decision tree).
   - Per-message custom logic at the edge (transform, validate, enrich, moderate) → serverless functions skill (see decision tree).

## Constraints

- **Never write implementation code in this skill.** Hand off to the specialist owner.
- **Never paraphrase another skill's content.** Link instead. The full mapping lives in [references/intent-to-tool.md](references/intent-to-tool.md).
- **Stop after one handoff.** If the user has multiple intents, ask which to address first.
- **Always name both the skill and the MCP tool.** A handoff without both is incomplete.
- **If the user has already named a specific feature**, do not run this skill — defer to the matching specialist directly.

## MCP Tools

This skill primarily lists tools rather than calling them. The full mapping is in [references/intent-to-tool.md](references/intent-to-tool.md). The most-frequently-routed-to tools on the `user-pubnub` MCP server are:

- **`get_chat_sdk_documentation`** — for any chat / messaging / typing-indicator / reaction / thread question.
- **`get_sdk_documentation`** — for any non-chat real-time question (core SDKs, IoT, state sync, notifications).
- **`how_to`** — for conceptual, step-by-step integration recipes.
- **`write_pubnub_app`** — for architecture review, production-readiness checks, and best-practices validation.
- **`manage_apps`**, **`manage_keysets`** — for environment setup and configuration questions.
- **`manage_app_context`** — for user/channel/membership metadata questions.
- **`manage_illuminate`** — for analytics, KPI, threshold-trigger, and dashboard questions.
- **`send_pubnub_message`**, **`subscribe_and_receive_pubnub_messages`**, **`get_pubnub_messages`**, **`get_pubnub_presence`** — for runtime testing and incident triage.

## See Also

This skill routes to every other skill in the catalog. The canonical owners it most often defers to:

- **pubnub-keyset-management** — for app/keyset/environment setup questions
- **pubnub-app-developer** — for non-chat SDK mechanics
- **pubnub-chat** — for chat-specific features
- **pubnub-presence** — for online/offline/occupancy questions
- **pubnub-history** — for replay, offline catch-up, and timetoken pagination
- **pubnub-app-context** — for user/channel/membership metadata
- **pubnub-functions** — for per-message edge logic
- **pubnub-events-and-actions** — for event-driven integrations to external systems
- **pubnub-illuminate** — for real-time analytics, decisions, and dashboards
- **pubnub-security** — for Access Manager, encryption, TLS
- **pubnub-scale** — for throughput, channel groups, large events
- **pubnub-reliability** — for backoff, idempotency, retry, dedup
- **pubnub-observability** — for logging correlation, testing, runbooks

## Output Format

When handing off, produce exactly one block in this shape:

```text
Intent      : <one sentence restating what the user is trying to do>
Skill       : <specialist-skill-name>
MCP tool    : <user-pubnub MCP tool>
Docs source : <which PubNub docs surface to consult>
Next step   : <one concrete action the user or agent should take next>
```

Then stop. Let the specialist skill take over.

## Canonical Owner Files

Every owned topic mentioned above defers to its canonical reference file. Use these links when handing off:

- Access Manager and PAM tokens → [pubnub-security/references/access-manager.md](../pubnub-security/references/access-manager.md)
- History timetoken pagination → [pubnub-history/references/pagination-and-ordering.md](../pubnub-history/references/pagination-and-ordering.md)
- Offline catch-up flow → [pubnub-history/references/offline-catch-up.md](../pubnub-history/references/offline-catch-up.md)
- App Context (users, channel metadata, membership metadata) → [pubnub-app-context/references/users.md](../pubnub-app-context/references/users.md)
- Message Actions and reactions → [pubnub-chat/references/message-actions.md](../pubnub-chat/references/message-actions.md)
- Channel groups and wildcard subscribe → [pubnub-scale/references/scaling-patterns.md](../pubnub-scale/references/scaling-patterns.md)
- Incident triage and runbook → [pubnub-observability/references/incident-runbook.md](../pubnub-observability/references/incident-runbook.md)
