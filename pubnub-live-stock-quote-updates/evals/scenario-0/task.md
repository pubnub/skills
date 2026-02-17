# Stock Quote Feed Publisher

## Problem/Feature Description

A financial data startup is building a backend service that ingests raw market data from an upstream WebSocket provider and redistributes it to client applications via PubNub. The upstream provider delivers data in an inconsistent format -- sometimes the price field is called `last`, sometimes `ltp`, sometimes `price`; ticker symbols may arrive as `sym`, `symbol`, or `ticker`. The service needs to handle the messy provider data, compute derived fields, and push updates out in a way that balances cost and latency for potentially hundreds of symbols updating multiple times per second.

The team wants a single Node.js module that connects to the upstream feed, processes incoming quotes, and publishes them through PubNub. During peak market volatility some symbols may produce dozens of updates per second, so the system needs to be smart about how frequently it sends full versus lightweight updates for each symbol.

## Output Specification

Produce the following files:

- `publisher.js` -- The main Node.js module that:
  - Initializes a PubNub connection for server-side publishing
  - Connects to an upstream WebSocket market data feed
  - Processes and normalizes incoming raw quote data into a consistent format
  - Computes derived fields where possible
  - Publishes updates to PubNub with appropriate per-symbol rate management
  - Handles reconnection if the upstream feed disconnects

- `package.json` -- Dependencies needed to run the publisher

No additional files are required.
