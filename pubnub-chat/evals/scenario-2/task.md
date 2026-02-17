# React Chat Application Shell

## Problem/Feature Description

A SaaS company is adding an in-app messaging feature to their React dashboard. They need a reusable chat foundation that any part of their application can hook into. The chat state (the initialized PubNub chat instance) should be accessible from deeply nested components without prop drilling. The architecture should handle the asynchronous nature of chat initialization, show loading and error states, and ensure that resources are properly released when the user navigates away or the app unmounts.

The team wants a ChatProvider component that wraps their app, a custom hook for accessing the chat instance, and a sample ChatRoom component that demonstrates connecting to a channel, displaying message history, and sending new messages. The chat room should clean up its subscriptions when the component unmounts or the channel changes.

## Output Specification

Create the following files:

1. `ChatProvider.jsx` - A React context provider component that initializes the PubNub Chat SDK, exposes the chat instance via context, and handles cleanup on unmount.
2. `useChat.js` - A custom hook that consumes the chat context.
3. `ChatRoom.jsx` - A component that takes a `channelId` prop, connects to that channel, loads message history, displays messages, and allows sending new messages. Must clean up subscriptions on unmount or channel change.
