# Social Feed Aggregator Service

## Problem/Feature Description

A social media platform lets users follow other users and topics. When a user follows someone or a topic, they should receive real-time updates from that source. A power user might follow hundreds of accounts and dozens of topics. The platform needs to efficiently subscribe each user to all of their followed sources without opening an excessive number of individual channel connections. Additionally, when a user opens the app after being offline, the platform needs to show them recent posts they missed.

The engineering team needs a JavaScript module that manages a user's feed subscriptions efficiently. The module should handle adding and removing followed sources from a user's feed, subscribing to all of them at once, and fetching missed messages when the user comes back online.

## Output Specification

Produce a single JavaScript file named `feed-aggregator.js` that exports:

1. A function to initialize the aggregator for a given user
2. A function to add a followed source (user or topic) to the user's feed
3. A function to remove a followed source from the user's feed
4. A function to start listening for real-time updates from all followed sources
5. A function to fetch recent missed messages from the user's feed channels after a period of being offline

Include inline comments explaining why certain PubNub features are chosen for managing large numbers of subscriptions.
