# Team Availability Dashboard

## Problem/Feature Description

A remote-first company wants a team availability dashboard that shows each team member's current status (available, busy, in a meeting, away) across all their devices. When someone updates their status on their phone, it should reflect immediately on the desktop dashboard. When someone closes all their browser tabs or their connection drops, they should show as offline. The dashboard is built as a single-page web application.

The company uses PubNub and each team has a channel (e.g., `team-{teamId}`). They need the client-side code that:
- Connects to the team channel with presence and sets the user's initial status
- Allows updating status dynamically
- Handles the user having multiple devices connected
- Ensures users are properly marked as offline when they close the browser

Build the client-side JavaScript module for this dashboard.

## Output Specification

Produce a file called `availability-dashboard.js` containing:

1. A function or class that initializes the PubNub client and connects to the team channel
2. Logic for setting and updating the user's custom status metadata
3. A presence event listener that tracks team members and their states
4. Proper lifecycle management so that closing the browser triggers the correct behavior
5. Handling for the case where a single user is connected from multiple devices

Also produce a file called `setup-notes.md` that documents:
- The PubNub client configuration choices and why they were made
- Any portal configuration that would be needed before this code works
