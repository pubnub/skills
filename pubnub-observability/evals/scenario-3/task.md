# Incident Triage Runbook for "Messages Are Being Dropped"

## Problem/Feature Description

The on-call engineer has been paged: support is reporting that messages published from the iOS app in the last hour are not showing up for subscribers on the same channel. The platform's incident response policy is that every triage starts from the documented runbook — no jumping to hypotheses, no skipping the four correlation fields. The team lead wants a standalone Markdown runbook that the on-call engineer can paste into the incident war-room channel and walk through step by step.

## Output Specification

Create a Markdown file called `messages-dropped-runbook.md`. The document must use these exact section headings, in this order:

```markdown
# Runbook: Messages Are Being Dropped

## Step 0: Collect Correlation Fields

## Step 1: Check PubNub Status Page

## Step 2: Pull Usage Metrics for the Window

## Step 3: Verify the Publish Actually Reached PubNub

## Step 4: Confirm the Message Landed in History

## Step 5: Check Receive-Side Dedup

## Step 6: Check Access Manager

## Step 7: Check Connection State

## Resolution

## War-Room Communication Template
```

Per-section requirements:

- **Step 0** must explicitly list the four correlation fields: `channel`, `message_id`, `user_id`, `timetoken`.
- **Step 1** must reference the PubNub status page URL `https://status.pubnub.com`.
- **Step 2** must name the `get_pubnub_usage_metrics` MCP tool and say to pull for the affected keyset and the spike's time window.
- **Step 3** must explain that the publisher's `pubnub.publish.ok` log line should include a `timetoken`; if only `pubnub.publish.start` is present, the publisher never got an ack.
- **Step 4** must name the `get_pubnub_messages` MCP tool and explicitly explain the two-way decision: in-history means publish-side succeeded and receive is the bug; not-in-history means publish-side has the bug OR the message was sent with `storeInHistory: false`.
- **Step 5** must mention dedup-by-message_id and direct the on-call to inspect the dedup set/LRU on the receiver.
- **Step 6** must mention `PNAccessDeniedCategory` in status logs.
- **Step 7** must mention `PNNetworkDownCategory` AND the short-term cache window for offline catch-up.
- **War-Room Communication Template** must contain a fenced code block that includes labelled lines for at least: INCIDENT, Started, Affected, Symptoms, Hypothesis, PubNub status page, Active correlation fields (channel + message_ids), Runbook in progress, On-call lead.

Do not include implementation code — this is a runbook, not a code module.
