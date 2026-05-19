# Define a COUNT Metric for Per-User Message Volume

## Problem/Feature Description

With the chat-message Business Object now in place, the trust-and-safety team needs a Metric that counts how many messages each user has published in a rolling 1-minute window. This Metric will be the source of a Decision that auto-mutes users above a configured threshold.

## Output Specification

Create a JavaScript file called `metric.js` that exports a function `createPerUserMessageCountMetric(businessObjectId, userIdFieldId)` returning a Promise that resolves to the parsed API response. The function must:

1. Read the API key from `process.env.ILLUMINATE_API_KEY`.
2. POST to `https://admin-api.pubnub.com/v2/illuminate/metrics` with the two required headers (Authorization, PubNub-Version: `2026-02-09`) plus Content-Type: application/json.
3. Send a JSON body that defines a Metric with:
   - `name`: `"Per-User Message Count (1m)"`
   - `businessObjectId`: the argument value
   - An aggregation of type COUNT (`function: "COUNT"` or equivalent shape used by the Illuminate API)
   - A grouping dimension on the user ID field whose source is `userIdFieldId`
   - `evaluationWindow`: 60 (one of the documented allowed values)
4. Parse the response as JSON and return it.

Use ES module syntax. Add a brief top-of-file comment noting that the only valid evaluation window values are 60, 300, 600, 900, 1800, 3600, and 86400 seconds.
