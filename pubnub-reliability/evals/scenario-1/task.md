# Idempotent Publish with Retry

## Problem/Feature Description

A chat client's incident review found that whenever the network glitches mid-publish, the retry loop creates duplicate messages — because each retry attempt regenerates the message UUID inline. The platform lead wants every publish in the codebase to use a single, well-tested helper that: creates the envelope once at the message-creation moment, embeds a stable message_id (UUID v4), retries with bounded backoff, and does NOT regenerate any envelope fields on retry.

## Output Specification

Create a JavaScript file called `idempotent-publisher.js` that exports the following:

1. `createMessage(type, payload)` - Returns a fully-formed envelope:
   ```
   {
     schema_version: 1,
     message_id: <UUID v4 from crypto.randomUUID()>,
     sent_at: <Date.now()>,
     type,
     payload
   }
   ```
   The `message_id` and `sent_at` are captured once at creation; the caller will not regenerate them.
2. `publishWithRetry(pubnub, channel, envelope, opts)` - Async. `opts` may include `{ maxAttempts: 5, initialMs: 200, maxMs: 30000, multiplier: 2 }`.
   - Loops at most `maxAttempts` times.
   - On each attempt, calls `pubnub.publish({ channel, message: envelope })`. The SAME envelope object (with its existing message_id) is passed on every attempt.
   - On success, returns the publish response.
   - On failure, computes a jittered exponential backoff (same algorithm as scenario 0) and awaits it before the next attempt.
   - After the final failed attempt, re-throws the last error.
   - The function MUST NOT mutate the envelope (no regenerating message_id, no updating sent_at).
3. `publishOnce(pubnub, channel, type, payload)` - A convenience that does `createMessage(type, payload)` and then `publishWithRetry(pubnub, channel, envelope)`.

Add a top-of-file comment explicitly stating:
- The message_id is generated once at envelope creation and never inside the retry loop.
- The retry strategy must use jittered backoff (not fixed interval).
- This producer pattern depends on receivers deduping by message_id; receivers are responsible for that.

Use ES module syntax.
