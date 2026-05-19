# User Profile Service for an Onboarding Flow

## Problem/Feature Description

A consumer fitness app is onboarding new users. As part of signup, the app needs to store a user profile in PubNub App Context — display name, email, an external CRM ID (so the marketing team can reconcile in their tool), an avatar URL, and a handful of preferences (timezone, units, notifications enabled). After signup, the profile screen needs to display all of this back to the user, and the user should be able to edit any field.

The lead engineer wants a small, well-tested user-profile module that the onboarding screen and the profile screen will both use.

## Output Specification

Create a JavaScript file called `user-profile.js` that exports the following functions:

1. `initPubNub(publishKey, subscribeKey, userId)` - Initializes a PubNub instance and returns it.
2. `createOrUpdateProfile(pubnub, userId, profile)` - Upserts a user profile in App Context. `profile` has shape `{ name, email, externalId, profileUrl, custom: { timezone, units, notifications_enabled, ... } }`. Returns the upsert result.
3. `getProfile(pubnub, userId)` - Reads a user profile from App Context, including all custom fields. Returns the user's full data including the `custom` object.
4. `listAdmins(pubnub, limit)` - Returns up to `limit` users whose `custom.role` equals `"admin"`, with custom fields included.
5. `deleteProfile(pubnub, userId)` - Removes the user object from App Context.

Use ES module syntax. Include a top-of-file comment noting that the App Context add-on must be enabled on the keyset.
