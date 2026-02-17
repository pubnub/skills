# Social Features for Community Chat

## Problem/Feature Description

An online learning platform is enhancing their student community chat. Students currently can send basic messages, but the product team wants richer interaction features. They need the ability to reply to specific messages in threads (so side discussions do not clutter the main feed), react to messages with emoji (for quick feedback on shared solutions), mention specific students by name to notify them, and reference other channels (such as linking to a course-specific channel from a general discussion).

The platform already uses PubNub Chat SDK for the base messaging. A developer needs to write a utility module that encapsulates these social features using the Chat SDK APIs. The module should work with any channel and message objects that are already instantiated.

## Output Specification

Create a JavaScript file called `social-features.js` that exports the following functions:

1. `openThread(message)` - Gets the thread for a given message object and returns it.
2. `replyInThread(thread, text)` - Sends a text reply within a thread.
3. `getThreadReplies(thread, count)` - Fetches the reply history for a thread.
4. `reactToMessage(message, emoji)` - Toggles an emoji reaction on a message.
5. `getReactions(message)` - Returns the reactions object from a message.
6. `sendWithMention(channel, text, mentions)` - Sends a message that mentions one or more users. The `mentions` parameter is an array of `{position, id, name}` objects indicating which mentions appear in the text.
7. `sendWithChannelRef(channel, text, refs)` - Sends a message that references one or more channels. The `refs` parameter is an array of `{position, id, name}` objects.
8. `loadHistory(channel, count)` - Loads message history for a channel with the given count.

Include JSDoc comments or inline comments explaining the expected shape of the parameters.
