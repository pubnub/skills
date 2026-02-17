# Player Safety and Regulatory Compliance Module

## Problem/Feature Description

A betting platform operating across multiple jurisdictions (UK, Malta, Gibraltar, Ireland, Sweden, Denmark) needs to implement a comprehensive player protection system. Regulators require that the platform enforce deposit limits, monitor session duration and net losses, support self-exclusion, and restrict access based on the player's geographic location.

The compliance team needs a system that runs checks before any betting activity is allowed and continuously monitors active sessions. When a player exceeds their configured limits, hits a session duration threshold, or accumulates losses beyond a safe level, the system must proactively alert them. Players who have opted into self-exclusion must be blocked from all betting channels entirely.

The platform also needs a geolocation verification layer that confirms the player is in a licensed jurisdiction before granting channel access, with periodic re-verification to guard against VPN circumvention.

## Output Specification

Produce the following files:

1. **`deposit-limits.js`** -- A module that:
   - Checks a user's deposit against their configured limits before allowing it
   - Tracks cumulative deposits per configurable time period
   - Returns structured responses indicating whether the deposit is allowed or blocked

2. **`session-tracker.js`** -- A JavaScript class that:
   - Tracks session start time, cumulative stakes, and cumulative winnings
   - Monitors net loss amounts
   - Sends alerts to the user via PubNub when configurable thresholds are exceeded
   - Implements a betting cool-down mechanism between rapid actions

3. **`self-exclusion-function.js`** -- A PubNub Function that:
   - Checks whether a user is self-excluded before allowing any betting action
   - Blocks access if the exclusion is active and has not expired
   - Returns an appropriate denial message with exclusion expiry information

4. **`geo-fence.js`** -- A geolocation verification system that:
   - Collects the user's coordinates and publishes them for verification
   - Includes a PubNub Function that checks coordinates against an allowed country list
   - Stores verification results with an expiry window for periodic re-verification
   - Handles the case where location access is denied
