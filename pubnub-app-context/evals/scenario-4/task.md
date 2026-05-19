# Live UI Reactions to Metadata Changes

## Problem/Feature Description

A community platform's web UI shows a sidebar with the user's joined rooms, each room's latest topic, and avatars of room members. The product team wants the sidebar to update live: when an admin renames a room, the new name should appear immediately; when a user updates their avatar, every visible instance should refresh; when a member is added or removed from a room, the membership badge should reflect that within a few seconds — all without a page reload.

The platform already uses PubNub for chat, and App Context Metadata Events are enabled on the keyset. The frontend team needs a small subscription utility that wires up listeners for user, channel, and membership changes and routes each event to the appropriate UI handler.

## Output Specification

Create a JavaScript file called `metadata-live.js` that exports the following:

1. `initPubNub(publishKey, subscribeKey, userId)` - Initializes a PubNub instance.
2. `subscribeToMetadataEvents(pubnub, handlers)` - Adds an objects listener and routes incoming events to the appropriate handler in `handlers`. The `handlers` object has the shape:
   ```
   {
     onUserSet, onUserDelete,
     onChannelSet, onChannelDelete,
     onMembershipSet, onMembershipDelete
   }
   ```
   Each handler receives the event's `data` object. Returns an unsubscribe function that removes the listener.
3. `watchUserChannels(pubnub, channelIds, userIds)` - Calls pubnub.subscribe to receive metadata events for the listed user IDs and channel IDs (recall that App Context metadata events are published on the corresponding object's channel — the userId or channelId itself).

Use ES module syntax.
