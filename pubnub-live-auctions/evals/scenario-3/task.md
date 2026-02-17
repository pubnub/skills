# Auction Room Client Application

## Problem/Feature Description

A sports memorabilia marketplace needs a JavaScript client application that provides the in-auction experience for bidders. The app must handle unreliable mobile networks gracefully -- bidders often participate from stadiums or events with poor connectivity, and dropped connections or failed bid submissions cannot mean losing an auction unfairly.

The client needs a smooth countdown timer that does not jump or stutter between updates from the server, a way to show how many people are currently watching the auction to build excitement, and a scrollable history of past bids. When the user navigates away from an auction room, the app must clean up its subscriptions to avoid wasting resources.

The client also needs to handle different failure modes when submitting bids -- showing the user a clear explanation if their bid was too low, if the auction has ended, or if they are already winning.

## Output Specification

Produce the following files:

1. **`auction-client.js`** -- The main client module that initializes the PubNub SDK, subscribes to the appropriate channels for an auction, and provides bid placement functionality with robust error handling.

2. **`countdown-display.js`** -- A client-side countdown timer class that renders a smooth timer display by working with periodic time updates from the server.

3. **`bid-history.js`** -- A module that retrieves and displays historical bids for an auction, loading past bids when the user first joins.

Include comments explaining the reconnection strategy, timer smoothing approach, and error handling decisions.
