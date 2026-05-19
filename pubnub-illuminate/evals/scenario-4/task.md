# METRIC Decision Limit Check Before Creation

## Problem/Feature Description

The platform engineering manager has rejected the last auto-merged change to Illuminate because it tried to create a fourth METRIC decision on an account that already has three — and the Illuminate API returned `400: "A business object cannot have more than 3 associated decisions."` only AFTER the rest of the resources had been provisioned. The team wants every future Decision-creation script to check the limit first, surface the conflict clearly to the user, and never delete an existing Decision without explicit confirmation.

## Output Specification

Create a JavaScript file called `safe-metric-decision.js` that exports the following functions:

1. `listMetricDecisions(apiKey)` - GETs `/v2/illuminate/decisions` with the required headers, parses the response, and returns the array of Decisions whose `sourceType === "METRIC"`.
2. `checkMetricDecisionLimit(apiKey)` - Returns `{ atLimit: boolean, count: number, decisions: Decision[] }` where `atLimit` is true when the METRIC decision count is `>= 3`.
3. `createMetricDecisionSafely(apiKey, decisionBody)` - The high-level helper. Behavior:
   - Calls `checkMetricDecisionLimit`.
   - If NOT at limit, proceeds to POST `decisionBody` to `/v2/illuminate/decisions` and returns `{ ok: true, decision }`.
   - If at limit, does NOT auto-delete. Returns `{ ok: false, reason: "limit-reached", atLimit: true, existingDecisions: <list> }` so the caller can prompt the user.
4. `deleteMetricDecisionWithConsent(apiKey, decisionId, confirmation)` - Only deletes when `confirmation === "yes-i-am-sure-delete-decision-" + decisionId`. Otherwise returns `{ ok: false, reason: "confirmation-required" }`. Includes a top-of-file comment explaining that cascade deletes are permanent and that this function is the only place in the module where DELETE is invoked.

All HTTP calls use the standard Authorization + PubNub-Version headers (`2026-02-09`).

Use ES module syntax. Add a top-of-file comment stating: METRIC Decisions are limited to 3 per account, and this module never auto-deletes — explicit per-call confirmation is required.
