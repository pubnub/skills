# PubNub Client Authentication Manager

## Problem/Feature Description

A team building a collaborative document editing tool needs a robust client-side JavaScript module that manages the PubNub connection lifecycle. Users connect to shared editing channels, and the app needs to handle authentication seamlessly -- fetching credentials from the backend, initializing PubNub, and gracefully handling situations where access is denied or credentials expire during a session.

The biggest pain point is that users currently get disconnected without warning when their session credentials expire, losing unsaved edits. The team wants a self-contained auth manager class that proactively refreshes credentials before they expire and handles access errors by automatically attempting to re-authenticate rather than dropping the connection.

Build this as a client-side JavaScript module that another developer can import and use.

## Output Specification

Create a file called `pubnub-auth-manager.js` containing:
- A class `PubNubAuthManager` that encapsulates all PubNub authentication logic
- A constructor that accepts a config object with `serverAuthUrl` (the backend endpoint URL) and `userSessionToken` (the current user's session JWT)
- An `initialize()` method that fetches credentials from the server, sets up PubNub, and starts credential lifecycle management
- Proper handling of authentication failures during the session
- A `subscribe(channels)` method that subscribes to the given channels
- A `destroy()` method that cleans up timers and disconnects
- The module should export the class as default

Also create a brief `usage-example.js` showing how to instantiate and use the class.
