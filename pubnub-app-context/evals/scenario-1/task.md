# Workspace Rooms and Memberships

## Problem/Feature Description

A team collaboration product organizes work into rooms (channels). Each user can belong to many rooms, and within a room they have a role (`owner`, `admin`, `member`, or `guest`). The product manager wants the engineering team to use PubNub App Context to track these relationships so the same data is queryable from any client.

The team needs a small membership module that: creates a room, adds a user to one or more rooms with role information attached to the membership (not to the room), lists the rooms a user belongs to with the room's display name and description hydrated inline, lists the members of a given room with their user info hydrated inline, and removes a user from a room.

## Output Specification

Create a JavaScript file called `workspace-membership.js` that exports the following functions:

1. `initPubNub(publishKey, subscribeKey, userId)` - Initializes a PubNub instance and returns it.
2. `createRoom(pubnub, roomId, name, description, custom)` - Upserts a channel object in App Context with the given metadata.
3. `addUserToRooms(pubnub, userId, rooms)` - Bulk-adds the user to many rooms in one call. `rooms` is an array of `{ id, role }` and the role is stored in the membership's custom field, not on the channel.
4. `listUserRooms(pubnub, userId)` - Returns the user's memberships. Each result should include the membership's custom (so the role is visible) AND the hydrated channel object's name, description, and custom fields, so the caller does not need a second round-trip per room.
5. `listRoomMembers(pubnub, roomId)` - Returns the room's members. Each result should include the membership's custom AND the hydrated user object's name and custom fields.
6. `removeUserFromRoom(pubnub, userId, roomId)` - Removes the user's membership in that room.

Use ES module syntax.
