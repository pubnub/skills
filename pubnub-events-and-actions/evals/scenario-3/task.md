# Presence Events to AWS Lambda via API Gateway

## Problem/Feature Description

A B2B platform wants to react to PubNub presence events (`user joined`, `user left`, `user timed out`) by updating a "currently online" badge in a Postgres database. The infrastructure team is already on AWS and wants the integration to go through API Gateway → Lambda, with PubNub configured to send a webhook with a shared-secret header that the Lambda validates before processing.

## Output Specification

Create a JavaScript file called `presence-lambda.js` that exports a Lambda handler:

```js
export const handler = async (event) => { ... };
```

The handler receives an API Gateway proxy `event`. It must:

1. Parse `event.body` (which is a JSON string) into an object.
2. Verify the request: read `event.headers['x-pubnub-secret']` (case-insensitive lookup is fine) and compare it against an expected secret loaded from `process.env.PUBNUB_SHARED_SECRET`. If it does not match, return `{ statusCode: 401, body: '' }` immediately.
3. Treat the parsed body as a batched PubNub E&A webhook payload — read `data` as the events array and `schema` as the event-type identifier.
4. Switch on the `schema` string and route each event:
   - schema contains `presence.user.channel.joined` → call exported `onJoin(event)`
   - schema contains `presence.user.channel.left` → call exported `onLeave(event)`
   - schema contains `presence.user.timedout` → call exported `onTimeout(event)`
   - anything else → call exported `onUnknown(event)`
5. Dedup each event by its `id` field against an in-memory `seen` Set so the same event id is not processed twice within a warm container.
6. Always return `{ statusCode: 200, body: '' }` after iterating the entire batch — never a non-2xx for partial processing failures inside the batch.

Also export `onJoin`, `onLeave`, `onTimeout`, `onUnknown`, and the `seen` set so tests can replace them.

Include a top-of-file comment that calls out:
- the shared-secret validation pattern,
- that PubNub may retry the same batch (so the Lambda is idempotent),
- that on a warm container the `seen` Set persists between invocations, providing best-effort dedup.

Use ES module syntax.
