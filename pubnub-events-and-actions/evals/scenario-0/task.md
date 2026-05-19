# Idempotent Webhook Receiver for Batched E&A Events

## Problem/Feature Description

A platform team uses PubNub Events & Actions to fan out chat-message events to an internal Node.js service that updates several downstream caches. The team is on a paid PubNub tier with batching enabled (default 100 items / 5 seconds) and retries set to 4. Because the receiver can see retried batches, the engineering manager wants the service to be safe against duplicate deliveries: the same event must never update the cache twice.

The team also wants the receiver to switch on the event's schema string so that different PubNub event types (message sent, message reaction created, message reaction deleted) call the right cache-update functions.

## Output Specification

Create a JavaScript file called `webhook-receiver.js` that exports an Express request handler `handleWebhook(req, res)`. The handler must:

1. Read the batched event list from `req.body.data` (defaulting to `[]` if absent).
2. Read the top-level `req.body.schema` string.
3. For each event in the batch, dedup using the event's `id` field against an in-memory `Set` (or similar) declared at module scope. If the id has been seen, skip the event without re-processing.
4. For un-seen events, route to one of three exported processor stubs based on the `schema` string:
   - `processMessageSent(event)` when the schema string contains `messages.message.sent`
   - `processReactionCreated(event)` when the schema string contains `messages.reaction.created`
   - `processReactionDeleted(event)` when the schema string contains `messages.reaction.deleted`
   - For any other schema, call `processUnknown(event)`.
5. After processing the entire batch, respond once with HTTP 200. Do not return a non-2xx for partial failures inside the batch.

Also export the processor stub functions (`processMessageSent`, `processReactionCreated`, `processReactionDeleted`, `processUnknown`) so they can be replaced in tests.

Use ES module syntax. Add a top-of-file comment noting that PubNub E&A may deliver the same batch more than once on retry and that the receiver must be idempotent.
