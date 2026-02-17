# Market Data Entitlement Service

## Problem/Feature Description

A global financial data platform serves users across four subscription tiers -- from free users who receive delayed quotes to professional traders who need full market depth. The platform covers US markets (NYSE and NASDAQ), the London Stock Exchange, and the Tokyo Stock Exchange. Different tiers of users should only be able to access the data channels they are entitled to, and the system must enforce this server-side rather than relying on client code.

The platform needs a server-side access control module that grants the correct PubNub channel permissions based on a user's subscription tier, and a utility module that resolves which channels to subscribe to for a given symbol and exchange. Free-tier users should see a visible notice that their data is delayed, including attribution to the data source. The channel architecture needs to cleanly separate different data tiers and different exchanges.

Build the server-side entitlement service and a client-side utility that determines the right channels and displays appropriate compliance information.

## Output Specification

Produce the following files:

- `entitlement-service.js` -- A server-side Node.js module that:
  - Grants PubNub channel access based on user subscription tier
  - Supports at least four tiers with different levels of data access
  - Handles users trading on multiple exchanges (US, London, Tokyo)

- `client-utils.js` -- A client-side utility module that:
  - Resolves the correct channel name for a given symbol and exchange
  - Provides appropriate disclaimer text based on the user's data tier and exchange
  - Renders data attribution information including delay status

- `package.json` -- Dependencies needed

No additional files are required.
