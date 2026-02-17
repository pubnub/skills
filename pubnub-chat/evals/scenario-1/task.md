# Team Collaboration Chat Room

## Problem/Feature Description

A project management tool needs a team chat feature where multiple team members can communicate in a shared channel. The product manager wants users to see when others are actively typing, similar to Slack or Discord. The engineering team has chosen PubNub's Chat SDK for the real-time layer.

The feature needs to support creating a team channel with several members, sending and receiving messages in real-time, and displaying typing indicators. The typing notification experience must feel responsive without overwhelming the PubNub service with too many signals. The application will have multiple chat rooms open simultaneously, so channel lookups need to be efficient and avoid redundant API calls.

## Output Specification

Create a JavaScript file called `team-chat.js` that exports the following:

1. `initTeamChat(publishKey, subscribeKey, userId)` - Initializes the chat SDK.
2. `setupUsers(chat, userList)` - Takes an array of `{id, name}` objects and ensures all users exist, returning user objects.
3. `createTeamChannel(chat, users, channelId, channelName)` - Creates a group conversation with the given members.
4. `getOrCreateChannel(chat, channelId, lookupFn)` - Retrieves a channel from a local cache, or calls lookupFn to create it and caches the result. Should use a module-level cache.
5. `connectWithTyping(channel, onMessage, onTypingChange)` - Subscribes to both messages and typing indicator events on a channel. Returns a cleanup function that unsubscribes from both.
6. `sendWithTypingStop(channel, text)` - Sends a message and stops the typing indicator.
7. `createTypingHandler(channel)` - Returns a function suitable for use as an input change handler that triggers typing indicators, but does not fire the signal excessively on rapid keystrokes.
