# Periodic Leaderboard Aggregation

## Problem/Feature Description

We are running a live polling application where users vote on items throughout the day. Votes are tracked using atomic counters in persistent storage with keys like `votes:item1`, `votes:item2`, etc. We need a PubNub Function that runs on a schedule (every 60 seconds) to collect the current vote tallies, compile them into a summary, and broadcast the leaderboard to subscribers on a `leaderboard` channel so that connected dashboards can update in real time. The broadcast should reach actual subscribers (not just internal function triggers), and the data should be available in message history.

The function also needs to read a configuration value from persistent storage (key: `poll_status`) to check whether the poll is still active. If the poll is inactive, the function should skip the broadcast and exit successfully.

## Output Specification

Produce the following files:

1. **`leaderboard-aggregator.js`** -- The complete PubNub Function source code.
2. **`deployment.md`** -- A deployment guide explaining how to set up this function, what trigger type and interval to use, and what to do before putting it into production.
