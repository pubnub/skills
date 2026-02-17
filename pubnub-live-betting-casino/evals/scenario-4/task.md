# Bet Settlement and Cash-Out Engine

## Problem/Feature Description

A sportsbook needs to build the post-placement lifecycle for bets: settling winning, losing, and voided bets, managing user balances, and offering live cash-out on in-play bets. When a match result is confirmed, the settlement engine must process all affected bets, update user balances, and notify each user in real time.

The platform supports single bets and accumulator bets with multiple selections. Accumulators are especially complex because a single voided leg changes the combined odds rather than voiding the entire bet. The settlement engine must handle this correctly.

For live bets, the platform also offers a cash-out feature that gives players the option to lock in a profit or minimize a loss before the event concludes. Cash-out values fluctuate with current odds and must be offered with a short validity window to avoid stale prices being accepted.

The balance system must reserve funds when a bet is placed and release or adjust them upon settlement, ensuring that the user's available and reserved balances remain consistent throughout the lifecycle.

## Output Specification

Produce the following files:

1. **`settlement-engine.js`** -- A module that:
   - Settles single bets (won, lost, void outcomes)
   - Settles accumulator bets with correct handling of voided legs
   - Updates user balances (available and reserved amounts)
   - Publishes real-time notifications to the user upon settlement

2. **`cashout-service.js`** -- A module that:
   - Calculates cash-out values based on original and current odds
   - Broadcasts cash-out offers to users with a validity window
   - Handles cash-out acceptance and processes the balance adjustment

3. **`balance-manager.js`** -- A module that:
   - Checks and reserves balance when a bet is placed
   - Releases reserved funds on settlement
   - Publishes balance updates to the user in real time
   - Handles concurrent bet placement safely using atomic operations

4. **`wager-lifecycle.js`** -- A module or set of constants/types that:
   - Defines the complete set of wager states a bet can be in
   - Provides transition logic or documentation showing valid state changes
   - Includes the standard bet rejection error codes used across the platform
