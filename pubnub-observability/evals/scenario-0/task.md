# Correlation-Logged Publish and Subscribe Wrappers

## Problem/Feature Description

A platform team's incident review identified that 80% of "messages stopped" investigations stall because the publish and subscribe code paths log inconsistently — some log a string, some log only `channel`, some forget the `timetoken` from the publish response. The lead engineer wants a small wrapper module that every service can drop in so the four documented correlation fields (channel, message_id, user_id, timetoken) are always logged, structured, on every send and every receive, plus connection-state events.

The team uses an internal structured logger called `log` exported from `./log.js` with the standard `log.info(event, ctx)` / `log.error(event, ctx)` shape.

## Output Specification

Create a JavaScript file called `pubnub-logged.js` that exports:

1. `publishWithLogging(pubnub, channel, envelope)` - Async. Logs a structured event `'pubnub.publish.start'` with `{ channel, message_id: envelope.message_id, user_id: pubnub.getUserId() }`, then awaits `pubnub.publish({ channel, message: envelope })`. On success, logs `'pubnub.publish.ok'` with the start context PLUS the `timetoken` from the publish response. On failure, logs `'pubnub.publish.error'` with the start context plus `error: err.message`. Returns the publish response (or re-throws).
2. `attachReceiveLogging(pubnub)` - Adds a `message` listener that, for each received message, logs `'pubnub.receive'` with `{ channel: e.channel, message_id: e.message?.message_id, user_id: pubnub.getUserId(), publisher_id: e.publisher, timetoken: e.timetoken }`. If the calling code's downstream processing throws, log `'pubnub.receive.process_error'` with the same context plus `error: err.message` (the wrapper accepts an optional `processFn(message)` argument; default no-op).
3. `attachStatusLogging(pubnub)` - Adds a `status` listener that logs `'pubnub.status'` with `{ category: s.category, operation: s.operation, affectedChannels: s.affectedChannels, lastTimetoken: s.lastTimetoken, currentTimetoken: s.currentTimetoken }`.

All logs MUST be structured (`log.info(event, ctx)` form, not string concatenation). The `publishWithLogging` and receive logs must NOT log the full message payload (PII risk); only the listed correlation fields.

Use ES module syntax. Import `log` from `'./log.js'`.
