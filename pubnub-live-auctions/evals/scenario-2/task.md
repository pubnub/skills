# Premium Auction Features: Reserve Prices, Anti-Sniping, and Notifications

## Problem/Feature Description

A luxury goods auction platform is upgrading their PubNub-based bidding system with three features their sellers and bidders have been requesting:

First, sellers want to set hidden minimum prices for their items. If bidding does not reach this threshold, the item should not be sold. Bidders should know whether the threshold has been crossed but should never learn the exact amount -- this protects the seller's negotiating position.

Second, bidders are frustrated by "sniping" -- last-second bids that win auctions before anyone can respond. The platform needs automatic timer extensions when bids arrive near the end of an auction, with sensible limits so auctions cannot run indefinitely.

Third, the previous high bidder needs to be notified immediately when someone places a higher bid, so they can decide whether to counter-bid. This notification must happen as part of the bid processing itself, not as a delayed background job, to keep the auction moving quickly.

## Output Specification

Produce the following files:

1. **`reserve-handler.js`** -- A PubNub Function (or module usable within one) that manages reserve price logic during bid processing. It should store, check, and broadcast reserve status without leaking the actual reserve amount.

2. **`auto-extend.js`** -- A module that implements the anti-sniping timer extension logic. It should determine when to extend, by how much, and when to stop extending. It should also produce the messages to broadcast to inform bidders about the extension.

3. **`outbid-notifier.js`** -- A module that sends notifications to the previous leading bidder when they are outbid. This should be designed to run within the bid validation PubNub Function.

Include comments in each file explaining the logic and constraints.
