# Paginated Ticket History Loader

## Problem/Feature Description

A B2B customer support platform stores every message in a ticket's channel using PubNub Message Persistence. Support agents frequently need to scroll back through long conversations — sometimes thousands of messages spanning weeks. The current UI fetches "the first 100 messages" and stops, which is frustrating for the team.

You are building the data layer for a new ticket history loader. The platform uses PubNub's JavaScript SDK for retrieval and needs a small, focused module that knows how to convert timetokens, page backward through history one bounded request at a time, and load all messages within a user-selected date range. The agent will reuse this module across the web app and a Node.js CLI debugging tool, so it must use ES module syntax and make no DOM assumptions.

## Output Specification

Create a JavaScript file called `history-pager.js` that exports the following functions:

1. `initPubNub(publishKey, subscribeKey, userId)` - Initializes a PubNub instance and returns it.
2. `timetokenToDate(timetoken)` - Converts a PubNub 17-digit timetoken (string) to a JavaScript Date.
3. `dateToTimetoken(date)` - Converts a JavaScript Date to a PubNub timetoken (string).
4. `fetchPage(pubnub, channel, options)` - Fetches one page of history. `options` may contain `count`, `start`, and `end`. Returns the array of messages for the channel (empty array if none).
5. `iterateHistory(pubnub, channel, startTimetoken)` - An async generator that yields one message at a time, walking backward through the channel's history from `startTimetoken` (or from the latest message if omitted) until the channel is exhausted.
6. `loadAllInRange(pubnub, channel, startTimetoken, endTimetoken)` - Fetches every stored message in `[startTimetoken, endTimetoken]`, paginating as needed, and returns the full array.

Use ES module syntax (import/export). Do not depend on any browser-only globals.
