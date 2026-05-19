# Offline Catch-up for a Mobile-Web Chat App

## Problem/Feature Description

A consumer chat app runs in mobile browsers, where users routinely lock their screens, switch apps, or lose connectivity in elevators and subways. The product manager wants users to see exactly what they missed when they come back — but only what's relevant. If a user has been offline for a week, replaying every message is pointless and expensive; one day's worth is plenty.

The platform uses PubNub with Message Persistence enabled. The engineering team needs a small JavaScript module that controls the SDK's reconnect-replay behavior explicitly, persists the last-seen timetoken per channel, runs a bounded catch-up only on actual reconnects, and merges those caught-up messages with the live stream without showing duplicates.

## Output Specification

Create a JavaScript file called `catch-up.js` that exports the following functions:

1. `initWithRestore(publishKey, subscribeKey, userId)` - Initializes a PubNub instance configured so the SDK's short-term cache replay is disabled. Returns the PubNub instance.
2. `saveLastSeen(channel, timetoken)` - Persists the last-seen timetoken for a channel using `localStorage`, keyed per channel.
3. `loadLastSeen(channel)` - Reads back the stored last-seen timetoken for a channel, or returns `null` if none is stored.
4. `catchUp(pubnub, channel, maxLookbackMs)` - Fetches every message between the stored last-seen timetoken and now, but never reaches further back than `maxLookbackMs` (i.e., if last-seen is older than `now - maxLookbackMs`, the lookback is capped to `maxLookbackMs`). Returns the messages in ascending timetoken order. Returns an empty array if there is no stored last-seen timetoken.
5. `setupCatchUpOnReconnect(pubnub, channel, maxLookbackMs, onMessage)` - Wires up a status listener so that catch-up runs only on actual reconnects (not on the initial connect). Also subscribes the PubNub instance to the channel and wires up a message listener. Caught-up messages and live messages are both deduplicated (by timetoken) and forwarded to `onMessage`. As each message is processed, the last-seen timetoken is updated.

Use ES module syntax. Assume `localStorage` exists in the target environment.
