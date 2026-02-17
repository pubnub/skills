# Live Collaboration User List

## Problem/Feature Description

A document collaboration platform (similar to Google Docs) needs to show who is currently viewing or editing a shared document. When a user opens a document, they should appear in the "active viewers" sidebar. When they close the tab or lose connection, they should be removed. The sidebar also shows a total count of viewers.

The platform uses PubNub for real-time features. Each document has a dedicated channel (e.g., `doc-{documentId}`). The team needs a JavaScript module that manages the list of active users for a given document channel. It must correctly handle all the ways users can appear and disappear, including network interruptions and high-traffic documents where hundreds of people might be viewing simultaneously.

Write the implementation for this presence-powered user list manager.

## Output Specification

Produce a file called `presence-manager.js` containing a JavaScript class (or module) that:

1. Maintains an in-memory map of currently active users in a channel
2. Sets up a presence event listener that processes all relevant event types
3. Provides a method to fetch the initial set of active users when the module starts
4. Provides methods to query the current user list and occupancy count
5. Includes callback hooks for UI updates when users join, leave, or change

The code should handle edge cases around network disruptions and be ready for channels of varying sizes.
