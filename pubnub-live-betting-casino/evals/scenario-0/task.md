# Live Football Odds Broadcasting Service

## Problem/Feature Description

A sports betting startup is launching a live football betting product. They need a backend service that broadcasts real-time odds updates to thousands of connected clients whenever a trader adjusts the prices for a match. The platform covers Premier League matches with markets like match winner, over/under goals, and both-teams-to-score.

The company has chosen PubNub as its real-time infrastructure and needs a JavaScript/Node.js module that handles: initializing the PubNub connection for the odds engine, organizing markets into a channel structure that clients can subscribe to efficiently, and publishing odds updates when the trading team pushes new prices. The service must support clients across the UK (fractional odds), Europe (decimal odds), and the US (American odds) simultaneously from a single broadcast.

During live matches, the service also needs to handle market suspensions triggered by in-game events such as goals or red cards -- suspending all markets for the affected match before any new odds are calculated and distributed.

## Output Specification

Produce the following files:

1. **`odds-engine.js`** -- A Node.js module that exports:
   - An initialization function that sets up the PubNub client for the odds engine
   - A function to set up channel groups for an event's markets
   - A function to publish an odds update for a given market and event
   - A function to suspend all markets for an event (given a reason) and then resume them with new odds
   - An odds conversion utility that converts between decimal, fractional, and American formats

2. **`client-subscriber.js`** -- A client-side module that:
   - Initializes a PubNub client for an end-user betting client
   - Subscribes to a football event's markets
   - Listens for odds updates and market suspensions
   - Handles connection error states (access denied, network issues, reconnection)

3. **`access-setup.js`** -- A server-side script that configures channel permissions:
   - Grants appropriate publish/subscribe permissions for the odds engine
   - Grants appropriate permissions for end-user clients
   - Sets up token-based authentication for individual users
