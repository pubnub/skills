# Full 4-Step METRIC Decision Workflow for Auto-Mute

## Problem/Feature Description

The Per-User Message Count Metric is in place. The trust-and-safety team now wants a Decision that auto-mutes any user who publishes more than 100 messages in a 60-second window. The action should set `muted: true` on the offending user's App Context metadata, and the Decision should only fire once per hour per offending condition group (so the same user isn't pummeled with repeated mute updates).

The team lead has read the Illuminate docs and knows Decisions require a 4-step workflow because rules reference auto-generated input/output/action IDs that are only available after the initial POST. They want a fully-automated module that drives the entire workflow end to end.

## Output Specification

Create a JavaScript file called `auto-mute-decision.js` that exports an async function `createAutoMuteDecision(metricId, businessObjectId, countFieldId, userIdDimensionFieldId)` returning the final enabled Decision response. The function must execute the full 4-step workflow:

1. **Step 1 — POST**: POST to `/v2/illuminate/decisions` with:
   - `sourceType: "METRIC"`, `sourceId: metricId`
   - `executionFrequency: 60`
   - `activeFrom`: an ISO-8601 UTC string in the past (e.g., `"2026-01-01T00:00:00Z"`)
   - `activeUntil`: an ISO-8601 UTC string well in the future (e.g., `"2027-12-31T23:59:59Z"`)
   - `hitType: "SINGLE"`
   - `executeOnce: false`
   - `enabled: false`
   - `rules: []`
   - `inputFields`:
     - One BUSINESSOBJECT input field for the count (since this is a COUNT metric): `name: "Message Count"`, `sourceType: "BUSINESSOBJECT"`, `sourceId: businessObjectId`, `dataType: "NUMERIC"`, `order: 1`
     - One DIMENSION input field for the user id: `name: "User ID"`, `sourceType: "DIMENSION"`, `sourceId: userIdDimensionFieldId`, `dataType: "TEXT"`, `order: 2`
   - `outputFields`: `[{ name: "Muted User", variable: "mutedUser" }]`
   - `actions`: a single `APPCONTEXT_SET_USER_METADATA` action with `actionValues` that include `executionLimitType: "ONCE_PER_INTERVAL_PER_CONDITION_GROUP"`, `executionInterval: 3600`, `userIdTemplate: "${User ID}"`, and a `customMetadata` object setting `muted: true`.
2. **Step 2 — read**: Parse the response and capture the auto-generated IDs from `inputFields`, `outputFields`, and `actions`.
3. **Step 3 — PUT with rules**: PUT to `/v2/illuminate/decisions/<id>` with the full body, this time including a `rules` array with a single rule:
   - `name: "Mute users sending >100 msgs/min"`
   - `conditionGroups`: one group with one condition `{ inputFieldId: <count-input-id>, operation: "NUMERIC_GREATER_THAN", value: "100" }`
   - `inputValues`: includes an entry for EVERY inputField — the count field with the same NUMERIC_GREATER_THAN/100 entry, and the user id field with `operation: "ANY", value: ""`
   - `actionIds`: `[<action-id>]`
4. **Step 4 — PUT enabled**: PUT the same full body again with `enabled: true`, and return the final response.

All calls use the same auth headers as the previous scenarios. Use ES module syntax.
