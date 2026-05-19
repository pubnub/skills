# Concurrent Profile Edits Without Clobbering

## Problem/Feature Description

A web app lets users edit their own profile in one tab while a desktop client is also signed in and may auto-sync small fields (theme, last-active timestamp) in the background. The platform has seen incidents where the background sync silently overwrites a partially-typed profile edit the user just submitted from the web. The engineering manager wants the profile editor to use App Context's optimistic concurrency support so the second writer either succeeds with a fresh read, or gets a clean conflict error and retries.

## Output Specification

Create a JavaScript file called `profile-editor.js` that exports the following:

1. `initPubNub(publishKey, subscribeKey, userId)` - Initializes a PubNub instance.
2. `readForEdit(pubnub, userId)` - Reads the user's profile with custom fields included, and returns the full data including the server-side `eTag`. The caller will keep the eTag and pass it back on save.
3. `saveWithETag(pubnub, userId, updates, eTag)` - Performs a setUUIDMetadata using the provided eTag for optimistic concurrency. `updates` is a partial profile (merge with existing). Returns `{ ok: true, data }` on success or `{ ok: false, reason: 'conflict' }` on eTag mismatch.
4. `saveWithRetry(pubnub, userId, updateFn, maxAttempts)` - High-level helper that: reads with eTag, calls `updateFn(currentData)` to compute the new value, attempts `saveWithETag`. If it conflicts, it re-reads and retries up to `maxAttempts` times. `updateFn` receives the current data and must return the new data including the merged custom object.

Use ES module syntax.
