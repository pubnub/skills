# Server-Side User Search with Pagination

## Problem/Feature Description

An internal moderation tool needs to search the company's users by name prefix and filter to those with verified status above a reputation threshold. The user base has grown to several hundred thousand records, so the team's prior approach — fetching every user and filtering in JavaScript — is no longer viable. The lead engineer wants all filtering and sorting pushed into App Context, with proper cursor pagination so the tool can lazy-load.

## Output Specification

Create a JavaScript file called `user-search.js` that exports the following:

1. `initPubNub(publishKey, subscribeKey, userId)` - Initializes a PubNub instance.
2. `searchUsersPage(pubnub, namePrefix, minReputation, cursor)` - Returns one page of users whose `name` matches the wildcard pattern `<namePrefix>*` AND whose `custom.verified` is `true` AND whose `custom.reputation` is `>= minReputation`. The page is sorted by name ascending, limit 100, with custom fields included. Returns `{ users, nextCursor }`.
3. `iterateMatchingUsers(namePrefix, minReputation, pubnub)` - An async generator that yields users page by page until the result is exhausted, using `searchUsersPage` under the hood.
4. `countMatching(pubnub, namePrefix, minReputation)` - Returns the total count of matching records (using App Context's `totalCount` include) without loading every record.

Use ES module syntax.
