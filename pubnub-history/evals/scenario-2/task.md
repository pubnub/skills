# Operations Dashboard Cross-Channel Activity Feed

## Problem/Feature Description

A site-reliability team runs an internal operations dashboard that consolidates the last 24 hours of activity across many PubNub channels: `incidents`, `deployments`, `support-pages`, `monitoring-alerts`, `release-notes`, and dozens of per-service channels. The current implementation issues one fetch per channel and stitches results in JavaScript, which is slow and chatty.

A staff engineer wants to rewrite this so it batches channels per call and sorts the merged stream by time. The catch: PubNub does not guarantee cross-channel ordering, and the per-channel message limit differs when fetching from a single channel versus multiple channels. The new module must respect both constraints and be safe to call with channel lists of any size, including more than the per-call channel limit.

## Output Specification

Create a JavaScript file called `multi-channel-history.js` that exports the following:

1. `init(publishKey, subscribeKey, userId)` - Initializes a PubNub instance.
2. `splitChannelBatches(channels, batchSize)` - Splits an array of channel names into batches of at most `batchSize` channels (default to PubNub's per-call channel maximum), returning an array of arrays.
3. `fetchMultiChannel(pubnub, channels, sinceTimetoken)` - Issues `fetchMessages` for the given channels using the correct per-channel limit based on whether `channels.length === 1` or `> 1`. Returns the raw `result.channels` object (channelName -> messages[]).
4. `mergeAndSort(channelResults)` - Flattens an object of `{ channelName: messages[] }` into a single array. Each message is augmented with its source `channel` property. Sorts the merged array in ascending order by timetoken.
5. `fetchAndMerge(pubnub, channels, sinceTimetoken)` - Convenience function that:
   - Splits the channel list with `splitChannelBatches`,
   - Calls `fetchMultiChannel` for each batch,
   - Merges all batch results,
   - Returns the final sorted, tagged array.

Use ES module syntax. Include a brief comment near `mergeAndSort` explaining that PubNub guarantees per-channel ordering only and that cross-channel ordering must be reassembled client-side.
