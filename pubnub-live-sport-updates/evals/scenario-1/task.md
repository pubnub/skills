# NBA Game Play-by-Play Client Module

## Problem/Feature Description

A basketball stats company is building a web application that shows a live play-by-play feed for NBA games. Fans open the app at various points during a game -- some from tip-off, others in the middle of the fourth quarter. The feed must show every play in the correct chronological order, even when network disruptions cause events to arrive out of sequence or when fans join after the game is already underway.

The company has noticed that during high-traffic playoff games, brief connection drops are common, and their current implementation loses events during those windows. They need a robust client-side module that maintains the timeline integrity regardless of network conditions, handles reconnection gracefully, and can catch up a late-joining user to the current game state.

## Output Specification

Produce a JavaScript file called `nba-play-timeline.js` that:

1. Initializes a PubNub client configured for a subscriber consuming an NBA game feed
2. Implements a timeline class that maintains an ordered list of play-by-play events, handling out-of-order arrivals and duplicate detection
3. Sets up a PubNub listener that routes incoming messages to the timeline by event type
4. Handles connection status changes (connected, disconnected, reconnected) and recovers missed events when the connection is restored
5. Includes a function to load recent game history for users who join a game already in progress
6. Exports or exposes the key classes/functions for use by the rendering layer
