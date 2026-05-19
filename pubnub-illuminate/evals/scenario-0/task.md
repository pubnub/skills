# Create a Business Object for Chat Message Analytics

## Problem/Feature Description

A trust-and-safety team has just enabled Illuminate on their production PubNub keyset. As the first step toward auto-mute on spam detection, they need a Business Object that extracts a handful of fields from every chat message: the publishing user ID, the channel name, the message text, and a numeric score the moderation function attaches under `body.moderation.score`.

The team's automation engineer needs a small Node.js helper that creates the Business Object via the Illuminate Admin API. The same helper will be reused in CI to provision Business Objects per environment.

## Output Specification

Create a JavaScript file called `business-object.js` that exports a function `createMessageBO(name)` returning a Promise that resolves to the API response JSON. The function must:

1. Read the API key from `process.env.ILLUMINATE_API_KEY` (do not accept it as an argument; do not hardcode it).
2. POST to `https://admin-api.pubnub.com/v2/illuminate/business-objects` using `fetch` (or the global `fetch` available in Node 18+).
3. Set both required headers: `Authorization` (with the value of `ILLUMINATE_API_KEY`) and `PubNub-Version` (with the value `2026-02-09`). Set `Content-Type: application/json`.
4. Send a JSON body with at least:
   - `name` (from the function argument)
   - `isActive: false` for now (so fields can be edited)
   - a `fields` array containing exactly these four field definitions, each with `name`, `jsonPath`, and `dataType`:
     | name | jsonPath | dataType |
     |---|---|---|
     | userId | `$.message.userId` | TEXT |
     | channel | `$.message.channel` | TEXT |
     | text | `$.message.body.text` | TEXT |
     | score | `$.message.body.moderation.score` | NUMERIC |
5. Parse the response as JSON and return it (or throw on non-2xx).

Use ES module syntax.
