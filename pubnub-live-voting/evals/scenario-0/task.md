# Live Conference Q&A Polling System

## Problem/Feature Description

A tech conference organizer is building a real-time audience polling feature for their annual developer summit. During keynote sessions, the host will present questions to the audience (e.g., "Which technology are you most excited about?"), and attendees will vote from their devices. The results need to update live on a large display screen behind the presenter.

The organizer needs a complete polling setup using PubNub that handles the full lifecycle of a poll: creating it with a set of options, opening it for votes, pausing it if needed during speaker transitions, closing it when the time is up, and finalizing the results. They also want late-joining attendees to be able to see what the current question and options are when they connect mid-session.

Build a Node.js module that provides the full poll setup and lifecycle management system. The conference will have an admin dashboard to control polls and an audience-facing client that subscribes to poll events.

## Output Specification

Produce the following files:

1. `poll-setup.js` - A module that exports functions for:
   - Initializing the PubNub SDK for both an admin and a participant client
   - Creating a poll configuration with a question, options, type, and scheduling
   - Opening, pausing, closing, and finalizing a poll
   - Setting up a participant client that listens for poll events and state changes

2. `channel-design.md` - A document listing the channels used by the system, their naming patterns, what each channel is for, and who publishes vs subscribes to each.

3. `example-usage.js` - A script demonstrating how an admin would create a poll with 4 options about favorite programming languages, open it, pause it, resume it, close it, and finalize it. Also show a participant client joining and listening for events.
