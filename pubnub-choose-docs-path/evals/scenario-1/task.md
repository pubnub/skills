# Disambiguate Functions vs Events & Actions

## Problem/Feature Description

A backend engineer at a marketplace company asks:

> "Every time someone publishes a chat message in our app, I want to scan it for obscenities and replace any flagged words with asterisks before any subscriber sees it. PubNub seems to offer both 'Functions' and 'Events & Actions' — which one should I be using for this? Don't write the code yet, just point me at the right thing."

The engineer explicitly wants a routing decision, not implementation. The key signal in the question is "before any subscriber sees it" — the moderation must happen inline, on each message, mutating the payload — which is the per-message edge-logic pattern, not the forward-to-external-system pattern.

## Output Specification

Produce exactly one handoff block in the canonical template format. Use exactly these five labeled lines (in this order, with the labels spelled exactly as shown):

```text
Intent      : ...
Skill       : ...
MCP tool    : ...
Docs source : ...
Next step   : ...
```

Do not include implementation code. Do not chain handoffs.
