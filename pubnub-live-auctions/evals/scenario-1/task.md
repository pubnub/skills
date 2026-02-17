# Auction Lifecycle and Timer Engine

## Problem/Feature Description

A collectibles auction house is moving from in-person events to an online platform powered by PubNub. They need a backend service that manages the full lifecycle of an auction -- from scheduling it ahead of time, through the active bidding window, to final completion. A key requirement is keeping countdown timers perfectly synchronized across hundreds of simultaneous viewers on different devices and networks, since their bidders have complained about timers being off by several seconds on competitor platforms.

The auction house runs multiple concurrent auctions, each with its own channel, and they want a unified catalog system so browsers can discover auctions that are active, upcoming, or recently completed. The system must handle the transition between normal bidding and a tense "closing" phase where the timer ticks down more rapidly for the audience.

## Output Specification

Produce the following files:

1. **`auction-lifecycle.js`** -- A server-side module that manages auction state transitions (creating, starting, transitioning between phases, and ending auctions). It should define which states exist and which transitions are valid, and publish appropriate events to the relevant channels when transitions occur.

2. **`countdown-broadcaster.js`** -- A server-side module that handles countdown timer broadcasting for active auctions. It should publish authoritative time-remaining updates to auction channels, adapting its broadcast behavior as the auction approaches its end.

3. **`catalog-manager.js`** -- A module that handles publishing auction listings to catalog channels so browsing users can discover auctions in different phases.

Include comments explaining the channel architecture and timing decisions.
