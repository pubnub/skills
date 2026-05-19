# Multiplayer Game Storage Policy

## Problem/Feature Description

A multiplayer card game uses PubNub for in-game state, lobby chat, and player presence. The team has Message Persistence enabled and they want chat messages stored for 30 days of catch-up, but they don't want every game state tick (player position deltas, hover cursors, "is typing" notifications) being written to storage — those flood the keyset with data the players never need to replay.

The lead engineer wants a single small storage-policy module that the rest of the codebase will use, so the rules are applied consistently. It should expose distinct publish helpers for the different storage tiers, a helper to count messages without paying to load them, and a verification helper used in CI / staging to confirm that a "stored" publish actually landed in history.

## Output Specification

Create a JavaScript file called `storage-policy.js` that exports the following functions:

1. `init(publishKey, subscribeKey, userId)` - Initializes and returns a PubNub instance.
2. `publishStored(pubnub, channel, payload)` - Publishes a message that should be stored in History.
3. `publishEphemeral(pubnub, channel, payload)` - Publishes a message that must not be stored in History (typing, cursors, deltas).
4. `sendSignal(pubnub, channel, payload)` - Sends a fully ephemeral signal using PubNub's signal API.
5. `countInWindow(pubnub, channels, sinceTimetoken)` - Returns the per-channel message counts since `sinceTimetoken` without loading any payloads.
6. `verifyStored(pubnub, channel, payload)` - Publishes `payload` to `channel`, waits long enough for storage to flush, fetches recent messages, and returns `true` if the published message can be retrieved by its publish-time timetoken; otherwise returns `false`.

Use ES module syntax. At the top of the file, add a brief comment block stating that messages sent via `publishEphemeral` and `sendSignal` are not retrievable from history.
