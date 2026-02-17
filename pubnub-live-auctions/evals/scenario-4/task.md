# Auction Completion, Permissions, and Activity Feeds

## Problem/Feature Description

A vintage car auction platform built on PubNub needs to finalize three areas of their system before launch.

First, when an auction ends, the platform must notify all relevant parties and update several systems: the winning bidder needs a personal notification, the catalog listings must be updated to reflect the auction's completion, and the auction result must be permanently recorded. If the seller had set a minimum acceptable price and it was not reached, the platform must not declare a winner.

Second, the platform needs proper access control. Auction administrators who can pause and cancel auctions must have different permissions than regular bidders. Bidders should be able to view and submit bids but should not have access to admin controls. Notification channels should be read-only for the recipient since only the server should write to them. All permissions should be time-limited.

Third, the platform wants a public activity feed for each auction that shows bid activity in real time, but bidder identities must be partially hidden for privacy. They also want analytics tracking that records bid statistics without slowing down the core bidding flow.

## Output Specification

Produce the following files:

1. **`auction-end.js`** -- A server-side module that handles the complete auction ending process, including all notifications and catalog updates that need to happen when bidding closes.

2. **`permissions.js`** -- A server-side module that configures channel access permissions for different user roles (bidders and administrators) across the various auction channels.

3. **`activity-feed.js`** -- A module that publishes bid activity events to the activity feed channel with privacy-preserving bidder identifiers.

4. **`analytics-handler.js`** -- A PubNub Function that tracks auction metrics (total bids, unique bidders, bid amounts) without interfering with the bid processing pipeline.

Include comments explaining the permission model and privacy approach.
