# Startup Accelerator Investment Voting Platform

## Problem/Feature Description

A startup accelerator runs a demo day where investors vote on which startups receive funding. The voting happens in multiple elimination rounds: after each round, the startup with the fewest votes is eliminated, and investors vote again among the remaining candidates until only the winner remains. Not all investors have equal voting power -- their vote weight corresponds to the size of their fund commitment (e.g., a lead investor's vote might count 5x, while an angel's vote counts 1x).

The accelerator needs a PubNub-based voting system that supports both the multi-round elimination workflow and weighted voting. Between rounds, the admin announces which startup was eliminated and opens the next round with the remaining options. Investor vote weights must be securely managed so that investors cannot inflate their own voting power.

Build the complete server-side logic for this multi-round, weighted voting system.

## Output Specification

Produce the following files:

1. `elimination-voting.js` - A module that manages multi-round elimination voting: tracking which round is active, which options remain, running a round, determining the lowest-scoring option, eliminating it, and advancing to the next round until a winner is determined.

2. `weighted-vote-processor.js` - A PubNub Function or module that processes incoming votes with per-voter weight multipliers. Weights are looked up server-side, never accepted from the client.

3. `round-admin.js` - Admin-side code for initializing voter weights before the first round, opening each new round, and publishing round transition events (eliminations, new round openings, final winner announcement).

4. `kv-schema.md` - A document describing the KV Store key structure used for round management, voter weights, and tallies.
