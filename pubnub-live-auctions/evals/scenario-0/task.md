# Auction Bid Processor

## Problem/Feature Description

An online art marketplace needs to add real-time bidding to their platform. They have chosen PubNub as their messaging infrastructure. The most critical piece is ensuring that bids are validated before other users see them -- the marketplace has had problems in the past with fraudulent and duplicate bids, race conditions when two people bid at the same instant, and users spamming tiny incremental bids to run up the count without meaningfully increasing the price.

The team needs a robust server-side bid processing pipeline that runs inside PubNub's serverless environment, along with the corresponding client-side bid submission code. The processor must handle all edge cases: concurrent bids, network-retry duplicates, bids on ended auctions, and bids from users who are already winning.

## Output Specification

Produce the following files:

1. **`bid-validator.js`** -- The server-side PubNub Function that intercepts and validates incoming bids before they reach subscribers. It should manage auction state, enforce bidding rules, and transform accepted bid messages.

2. **`bid-client.js`** -- The client-side JavaScript module that provides a `placeBid(pubnub, auctionId, amount)` function for submitting bids, including the bid message format used when publishing.

3. **`increment-rules.js`** -- A module (usable by the server-side function) that determines the minimum bid increment based on the current auction price. The marketplace wants a tiered system where higher-priced items require larger minimum jumps to prevent penny-bidding.

Include comments in each file explaining the design decisions.
