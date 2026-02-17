# Personalized News Feed with Offline Support

## Problem/Feature Description

A social news aggregation app lets users subscribe to topic feeds (e.g., technology, sports, finance, politics). Each user follows a different combination of topics and also receives personal notifications and direct messages. Users frequently go offline (commute, sleep) and when they come back online, they need to catch up on messages they missed. Some messages are ephemeral system announcements (like "server maintenance in 5 minutes") that should not clutter the history. Users should also be able to "delete" posts from their view, but the original content must be preserved for moderation purposes.

The app uses PubNub (JavaScript SDK, Node.js) for the real-time layer. The team needs the feed management backend and the client-side reconnection logic.

## Output Specification

Produce the following files:

- `feed-manager.js`: A Node.js module that exports:
  - A function to create a personalized feed subscription for a user given their user ID and list of topic subscriptions
  - A function to add a new topic to a user's feed
  - A function to remove a topic from a user's feed
  - A function to list all channels currently in a user's feed

- `feed-client.js`: A Node.js module that exports:
  - A function to initialize the client and load recent history for the user's feed channels
  - A function to handle reconnection after being offline, fetching any messages that were missed
  - A function to publish an ephemeral system announcement that should not be stored in history
  - A function to "delete" a message from the user's view (for moderation-safe deletion)
  - A function to load history that properly filters out deleted messages

- `setup-guide.md`: A brief document describing what PubNub features and add-ons need to be enabled in the Admin Portal for this system to work, and any default behavior the team should be aware of regarding message storage and buffering.
