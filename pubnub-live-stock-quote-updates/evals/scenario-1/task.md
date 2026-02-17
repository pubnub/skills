# Real-Time Portfolio Tracker

## Problem/Feature Description

A retail brokerage wants to add a real-time portfolio dashboard to their web application. Each user holds a set of stock positions (symbol, number of shares, and average cost basis). Users also maintain a personal watchlist of symbols they want to monitor beyond their holdings. When a user opens the dashboard, they should immediately see the most recent prices for all their positions and watchlist symbols, and then receive live updates as prices change.

The dashboard needs to show per-position gain/loss (both total and for the current day), overall portfolio value, and an indication of how fresh the displayed data is. Users frequently add and remove symbols from their watchlists, and the system should handle these changes without requiring a full reconnect. When a user removes a symbol, the system should not continue receiving updates for it.

Build a client-side JavaScript module that manages watchlist subscriptions and portfolio calculations using PubNub as the real-time transport layer.

## Output Specification

Produce the following files:

- `portfolio-tracker.js` -- A JavaScript module that:
  - Manages a user's watchlist subscriptions through PubNub
  - Handles adding and removing symbols dynamically
  - Fetches initial quote data when the user first loads the dashboard
  - Calculates per-position and total portfolio gain/loss values in real time as quotes arrive
  - Detects and flags when quote data may be outdated
  - Properly handles both full quote updates and lightweight price tick updates

- `package.json` -- Dependencies needed

No additional files are required.
