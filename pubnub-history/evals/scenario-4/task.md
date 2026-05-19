# GDPR Right-to-Be-Forgotten Worker

## Problem/Feature Description

A consumer messaging product is built on PubNub with long-term Message Persistence. When a user submits a "delete my data" request under GDPR, the platform's backend must permanently remove that user's messages from a specific channel within a given time window — and then confirm the deletion succeeded.

The platform's security team has been clear that this code runs server-side only, never in a client. PubNub's deletion API has specific keyset and key-type requirements that must be respected. A senior backend engineer is writing this as a small Node.js worker that runs on a job queue, processing one GDPR request at a time.

## Output Specification

Create a JavaScript file called `gdpr-deletion.js` that exports the following:

1. `initServerPubNub(publishKey, subscribeKey, secretKey, userId)` - Initializes a server-side PubNub instance that is authorized to call `deleteMessages`. Returns the PubNub instance.
2. `deleteRange(pubnub, channel, startTimetoken, endTimetoken)` - Deletes all stored messages on `channel` in the half-open range `(start, end]`. Returns the API response.
3. `deleteUserMessages(pubnub, channel, userId, fromTimetoken, toTimetoken)` - Fetches messages from the channel in the bounded window, filters to those whose `uuid` matches `userId`, and deletes those messages (paginating as needed). Returns the count of messages deleted.
4. `confirmDeletion(pubnub, channel, startTimetoken, endTimetoken)` - Re-fetches the channel range after deletion and returns `true` if no messages remain in the range, otherwise `false`.

At the top of the file, include a clear comment block stating:
- This module must only run server-side.
- The secret key must never appear in client-side code.
- The PubNub keyset must have the "Delete from History" feature enabled before any of these functions will succeed.

Use ES module syntax.
