# Route a Runtime Debugging Question

## Problem/Feature Description

An on-call engineer pings you during an incident:

> "We're getting reports that messages from earlier today aren't showing up for users in the support channel. I just need to look at what's actually in the channel right now — like, what messages are stored for the last hour on `support-incidents-prod`. Don't write code, just tell me what to use."

This is a runtime / incident-triage question, not a design question. The user wants to inspect what is stored in PubNub right now. The right route is the History specialist with the runtime MCP tool that fetches stored messages directly.

## Output Specification

Produce exactly one handoff block in the canonical template format:

```text
Intent      : ...
Skill       : ...
MCP tool    : ...
Docs source : ...
Next step   : ...
```

The Next step should give a single actionable suggestion appropriate for an active incident (e.g., "Use get_pubnub_messages on the support-incidents-prod channel for the last hour to confirm storage"). Do not include implementation code.
