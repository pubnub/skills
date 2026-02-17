# Bet Validation and Placement System

## Problem/Feature Description

A regulated online sportsbook needs to build the server-side bet validation pipeline for their platform. When a customer places a bet through the mobile app, the wager is submitted to a PubNub channel. Before the bet reaches the backend settlement engine, it must pass through a series of validation checks running as serverless handlers.

The validation pipeline needs to catch malformed bets, enforce stake boundaries, verify that submitted odds have not drifted too far from the current market prices, and reject bets where the customer does not have sufficient funds. The team also needs a client-side bet slip component that assembles single bets and combination bets (accumulators) before submission.

The entire validation chain must operate in near real-time, returning acceptance or rejection notifications back to the specific user who placed the bet. The platform must guard against automated abuse through submission throttling.

## Output Specification

Produce the following files:

1. **`bet-validation-function.js`** -- A PubNub Function (Before Publish handler) for the wager submission channel that:
   - Validates the structure of incoming bet messages
   - Enforces stake limits
   - Enforces selection count limits for accumulator bets
   - Stamps validated bets with a server timestamp and status

2. **`odds-verification-function.js`** -- A PubNub Function (After Publish handler) that:
   - Checks submitted odds against stored current prices
   - Detects when odds have shifted beyond an acceptable threshold
   - Sends acceptance or rejection notifications back to the user on their personal status channel
   - Includes appropriate error codes in rejection messages

3. **`bet-slip.js`** -- A client-side JavaScript class that:
   - Manages a bet slip with one or more selections
   - Supports single bets and accumulator bet types
   - Calculates potential returns
   - Submits bets to the wager channel with all required fields
   - Handles price locking for selected odds with an expiry mechanism

4. **`bet-status-listener.js`** -- A client-side listener that subscribes to the user's status channel and handles different response types (accepted, rejected, odds changed, settled, cash-out offers).
