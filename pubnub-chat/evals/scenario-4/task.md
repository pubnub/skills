# Notification System for Multi-Channel Chat

## Problem/Feature Description

A customer support platform manages many concurrent chat channels: public FAQ rooms, private support tickets, and internal team channels. Support agents need a notification center that shows which channels have unread messages and when the last message was received in human-readable time. When an agent opens a channel, it should be marked as read so the unread badge clears.

The platform uses PubNub Chat SDK. A developer needs to build a notification utility module that handles unread tracking, read receipt management, and time display formatting. The module should also support creating a public announcement channel that any agent can browse, since some channels are open to all support staff.

## Output Specification

Create a JavaScript file called `notification-utils.js` that exports the following functions:

1. `initNotifications(publishKey, subscribeKey, userId)` - Initializes the chat SDK and returns the chat instance.
2. `createAnnouncementChannel(chat, channelId, name, description)` - Creates a public channel that any user can join for team-wide announcements.
3. `getUserChannels(chat)` - Returns a list of channels the current user is a member of, using the membership API.
4. `getUnreadCount(channel)` - Returns the number of unread messages for a given channel.
5. `markAsRead(channel)` - Marks a channel as read up to the most recent message.
6. `formatMessageTime(message)` - Takes a message object and returns a human-readable time string by converting the message's timetoken.
7. `getRecentMessages(channel, count)` - Fetches recent messages for a channel.
8. `buildNotificationSummary(chat)` - Returns an array of objects, each with channelId, channelName, unreadCount, and lastMessageTime (formatted) for all channels the user belongs to.

Use ES module syntax.
