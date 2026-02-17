# Live Auction Bidding Component

## Problem/Feature Description

An online auction platform is building a real-time bidding feature as a React component. When a user views an auction item, the component should connect to a channel for that specific auction and display incoming bids in real time. Users should also be able to place bids through the same component. The platform has experienced memory leaks in production because previous implementations did not properly clean up PubNub connections when users navigated away from auction pages.

The team needs a React component that manages the full PubNub connection lifecycle correctly. The component receives the current user's ID and the auction item ID as props, connects to the appropriate channel, displays real-time bid updates, and allows the user to submit new bids. It must handle navigation between different auction items without leaking connections.

## Output Specification

Produce a single React component file named `AuctionBidPanel.jsx` (or `.tsx`) that:

1. Accepts `userId` and `auctionId` as props
2. Connects to a PubNub channel for the given auction and listens for bid messages
3. Displays incoming bids in a list
4. Provides a form input for the user to submit a new bid amount
5. Properly manages the connection lifecycle when the component mounts, updates (e.g. different auctionId), and unmounts
