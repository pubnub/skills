# Live + History Merge with Dedup

## Problem/Feature Description

A chat app's clients fetch the recent message history when they open a room, then subscribe live to that channel. Because the history fetch overlaps with newly-live messages, users have been complaining that some messages appear twice in the UI. The platform engineering lead wants a small utility that handles the live + history merge correctly: catch up first, then subscribe live, with a per-channel bounded LRU keyed by message_id so any overlap is filtered out before reaching the UI.

## Output Specification

Create a JavaScript file called `live-history-merge.js` that exports the following:

1. A simple `LRU(max)` class with `has(key)` and `add(key)` methods. Both methods update recency so the least-recently-used entry is the one evicted when `add` exceeds `max`. Default `max` is 10000.
2. `createDedupRegistry()` - Returns an object with `has(channel, key)` and `add(channel, key)` methods. Internally maintains one `LRU` per channel (created lazily on first use). The same key on two different channels does NOT dedup against each other.
3. `mergeHistoryThenSubscribe(pubnub, channel, registry, onMessage)`:
   - First fetch the most recent N=100 messages from `pubnub.fetchMessages({ channels: [channel], count: 100 })`.
   - For each fetched message, in chronological order, compute `key = message.message_id ?? message.timetoken`, check the registry; if not seen, add it and call `onMessage(message)`.
   - THEN attach a `message` listener with `pubnub.addListener({ message: (e) => ... })` that does the same dedup-then-onMessage flow for each live message.
   - THEN call `pubnub.subscribe({ channels: [channel] })`.
   - It is critical that the catch-up runs to completion BEFORE the live subscription starts — so that any live messages whose ids overlap with the catch-up are filtered by an already-populated registry.

Add a top-of-file comment explicitly stating:
- Process catch-up FIRST, then subscribe live (order matters).
- Prefer message_id over timetoken as the dedup key; timetoken is a fallback.
- The dedup registry is per-channel; one global set across channels would conflate independent streams.

Use ES module syntax.
