# Author JSONPath Filters for an E&A Listener

## Problem/Feature Description

A multi-tenant SaaS platform routes PubNub events to multiple downstream systems via Events & Actions. The platform engineer needs to author several Advanced JSONPath filters that scope different listeners narrowly. The filters will be pasted into the Admin Portal as-is, so each must be a complete, syntactically valid JSONPath filter expression — wrong syntax silently drops every event with no error message.

## Output Specification

Create a JavaScript module called `filters.js` that exports a single named constant `FILTERS`. `FILTERS` is an object whose values are JSONPath filter strings, one per use case below. Each value must be a single string containing a complete JSONPath filter expression (no surrounding code, no template literals interpolating runtime values — pure static expressions).

The keys and their meanings:

1. `tenantOnly` — Triggers only on channels whose name starts with `t42.` (case-insensitive).
2. `highPriorityChat` — Triggers only when the channel name starts with `chat.` (case-insensitive) AND the message payload has `priority` equal to `"high"`.
3. `sensorThreshold` — Triggers only when the channel is exactly `sensors` AND the publish metadata's `reading` field is greater than `80`.
4. `excludeSystem` — Triggers on every event EXCEPT those where the message payload has `system` equal to `true`.
5. `botOrAdmin` — Triggers only when the publisher's `uuid` is in the array `["bot-system-1", "bot-system-2", "admin-1"]`.

Also export a brief Markdown string named `NOTES` that explains in two or three lines that JSONPath filters are tested before saving because wrong syntax silently drops every event.

Use ES module syntax.
