# Secure Chat Room with Presence

## Problem/Feature Description

A healthcare messaging application requires strict access control on all channels. The app uses PubNub's Access Manager to ensure only authorized users can read and write messages. Each consultation room has a channel (e.g., `consult-{roomId}`), and the application needs to show doctors and patients who is currently in the room.

The application must handle unreliable network conditions gracefully -- medical staff move between areas with poor connectivity, so the client needs robust reconnection logic that keeps the presence state accurate after a network drop. The heartbeat configuration needs to be tuned for a real-time chat scenario where quick detection of disconnected users matters.

Build the server-side access grant logic and the client-side presence module for this application.

## Output Specification

Produce two files:

1. `server-grants.js` -- A Node.js script that demonstrates how to grant the necessary permissions for a user to join a consultation room channel with full presence capabilities.

2. `client-presence.js` -- The client-side JavaScript module that:
   - Initializes the PubNub client with appropriate configuration for real-time chat
   - Subscribes to the consultation channel with presence
   - Handles network connectivity changes robustly so users see accurate room status even after drops
   - Loads the current room occupancy when appropriate
   - Sets up a presence listener for the channel
   - Configures heartbeat and timeout values appropriate for real-time chat
