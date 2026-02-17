# NFL Score Ingestion Service

## Problem/Feature Description

A sports media startup is building a backend service that receives raw scoring data from their NFL data provider and re-broadcasts it to their PubNub-powered client applications. The data provider sends webhooks whenever a scoring play occurs (touchdowns, field goals, safeties) as well as non-scoring events (penalties, turnovers, quarter changes).

The service needs to normalize these raw events and publish them to PubNub so that multiple different client applications -- a live scoreboard, a play-by-play feed, and a league-wide score ticker -- can each subscribe to exactly the data they care about without receiving unnecessary traffic. The service must also ensure that events can be reliably ordered by consumers even if network conditions cause out-of-order delivery.

## Output Specification

Produce a single JavaScript file called `nfl-ingestion-service.js` that:

1. Initializes the PubNub SDK for use as a server-side service
2. Implements a class or module that accepts raw game events, normalizes them, and publishes them to the appropriate PubNub channels
3. Handles both scoring plays (touchdowns, field goals, safeties) and non-scoring plays (penalties, turnovers, individual plays)
4. Includes a validation function that checks incoming events before publishing
5. Supports publishing updates for multiple concurrent NFL games

The code should be a complete, runnable Node.js module (assuming PubNub credentials are available as environment variables).
