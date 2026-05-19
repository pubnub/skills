# BUSINESSOBJECT Decision for Per-Message Spam Score

## Problem/Feature Description

The trust-and-safety team wants a second Decision that fires per individual message (not on aggregated counts). When any single message has `body.moderation.score` greater than `0.9`, they want a webhook called immediately, passing the offending message's userId and channel. This is a BUSINESSOBJECT-source Decision (the source is the BO itself, not a Metric or Query).

This source type has several documented gotchas the engineering manager wants the implementation to get right on the first attempt:

- BUSINESSOBJECT decisions require BOTH `sourceId` and `businessObjectId` set to the same BO ID.
- `inputFields` for BUSINESSOBJECT decisions use sourceType `"FIELD"`, NOT `"DIMENSION"`.
- `executionFrequency` must be **omitted entirely** — sending `null` triggers a plan error.

## Output Specification

Create a JavaScript file called `per-message-spam-decision.js` that exports an async function `createSpamWebhookDecision(businessObjectId, scoreFieldId, userIdFieldId, channelFieldId, webhookUrl)` returning the final enabled Decision response. The function must execute the 4-step workflow:

1. **Step 1 — POST**: POST to `/v2/illuminate/decisions` with:
   - `sourceType: "BUSINESSOBJECT"`
   - BOTH `sourceId: businessObjectId` AND `businessObjectId: businessObjectId` (same value)
   - DO NOT include `executionFrequency` at all (omit the property, do not set to null)
   - Required Decision fields (`activeFrom`, `activeUntil`, `hitType: "SINGLE"`, `executeOnce: false`)
   - `enabled: false`
   - `rules: []`
   - `inputFields`: three entries, all with `sourceType: "FIELD"`:
     - `{ name: "Score", sourceType: "FIELD", sourceId: scoreFieldId, dataType: "NUMERIC", order: 1 }`
     - `{ name: "User ID", sourceType: "FIELD", sourceId: userIdFieldId, dataType: "TEXT", order: 2 }`
     - `{ name: "Channel", sourceType: "FIELD", sourceId: channelFieldId, dataType: "TEXT", order: 3 }`
   - `outputFields`: `[{ name: "Offender User", variable: "offenderUser" }, { name: "Offender Channel", variable: "offenderChannel" }]`
   - `actions`: one `WEBHOOK_EXECUTION` action with `actionValues` containing `url: webhookUrl`, `executionLimitType: "ALWAYS"`, and a `payload` template that includes `${User ID}` and `${Channel}` interpolations.
2. **Step 2**: parse the POST response and read auto-generated IDs.
3. **Step 3 — PUT with rules**: include a single rule whose `conditionGroups[0].conditions[0]` is `{ inputFieldId: <score-input-id>, operation: "NUMERIC_GREATER_THAN", value: "0.9" }`, and whose `inputValues` covers all three input fields (score with the NUMERIC_GREATER_THAN/0.9 entry; user id and channel each with `operation: "ANY", value: ""`). PUT the body to `/v2/illuminate/decisions/<id>` (still WITHOUT `executionFrequency` and still with both `sourceId` and `businessObjectId`).
4. **Step 4 — PUT enabled**: PUT again with `enabled: true`.

Use ES module syntax.
