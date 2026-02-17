# Secure Vote Processing for a Company Board Election

## Problem/Feature Description

A corporate governance platform needs to run a secure board of directors election using PubNub. Employees submit votes from their workstations, but the company has strict integrity requirements: no duplicate voting, no voting after the poll closes, and no invalid ballot options. All vote validation must happen server-side so that a tampered client cannot stuff the ballot box.

The engineering team needs a PubNub Function that intercepts every incoming vote before it reaches any subscribers. The function must check that the vote contains required fields, verify the poll is currently accepting votes, prevent the same person from voting twice, confirm the selected option is valid, and enforce the poll's closing time. Invalid votes should be rejected with clear error codes, while valid votes should update the tally and allow the message through.

Build the server-side vote processing pipeline as a PubNub Function, along with a helper module that initializes the poll state needed for the function to operate.

## Output Specification

Produce the following files:

1. `vote-handler.js` - A PubNub Before Publish Function that intercepts votes, validates them, prevents duplicates, checks timing, and processes valid votes. This is the core server-side logic.

2. `poll-init.js` - A module that exports a function to initialize all required server-side state for a new poll before it opens (setting status, registering valid options, zeroing counters, etc.).

3. `error-reference.md` - A reference document listing all the error codes the vote handler can return, what triggers each one, and what the client should do in response.
