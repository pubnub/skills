# Real-Time Tally Dashboard for a TV Talent Show

## Problem/Feature Description

A television production company is building a live voting experience for their talent show finale. Millions of viewers will vote for their favorite contestant from home during a 5-minute voting window. As votes come in, the production team needs a live tally to display on-air in real time, showing each contestant's vote count and percentage updating as new votes arrive.

The system faces two key challenges: first, the vote tallying must be accurate even under extremely high concurrency -- thousands of votes arriving every second cannot cause lost counts. Second, the result broadcasting needs to be throttled so the on-air display updates smoothly without being overwhelmed by a firehose of updates.

The show also wants to support a "change your vote" feature during the voting window -- if a viewer changes their mind before the poll closes, their old vote should be removed from one contestant's count and added to another, keeping the total accurate.

Build the server-side tallying logic and the client-side result display subscription for this system.

## Output Specification

Produce the following files:

1. `tally-function.js` - A PubNub Function that processes valid votes, updates tallies, handles vote changes, and broadcasts consolidated result updates to viewers. Include logic to handle high-throughput scenarios.

2. `results-client.js` - A client-side module that subscribes to result updates and provides functions for rendering live tallies. Show how to aggregate the incoming data into percentages and rankings.

3. `vote-change-handler.js` - A module or function demonstrating how a voter can change their vote and how the server correctly adjusts the tallies for both the old and new option.
