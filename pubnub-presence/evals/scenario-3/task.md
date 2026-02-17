# Live Event Audience Tracker

## Problem/Feature Description

A live-streaming platform needs to show audience counts for ongoing events. Some events are small (a few dozen viewers) and should display a full list of who is watching. Other events are massive, with tens of thousands of concurrent viewers, where only an aggregate count is practical. The platform also has a lobby channel where all users are present but presence tracking is not needed -- only the individual event channels need it.

The platform uses PubNub with channels like `event-{eventId}` for each live stream and a `lobby` channel for general messaging. The engineering team needs a JavaScript module that intelligently handles presence at different scales, avoids unnecessary overhead, and provides cached occupancy data for the dashboard that refreshes every few seconds.

Write the implementation for the scalable presence tracking module.

## Output Specification

Produce a file called `audience-tracker.js` containing a JavaScript class or module that:

1. Subscribes to multiple channels but is selective about which ones get presence tracking
2. Handles presence events appropriately for both small and large audience channels
3. Provides a method to get the current occupancy of a channel, with caching to avoid redundant API calls
4. Includes the PubNub client initialization with appropriate configuration

Also produce a file called `scaling-notes.md` that documents:
- How the module handles different audience sizes
- What portal-level configuration would be needed for presence at scale
- Performance tuning recommendations for large channels
