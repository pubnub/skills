# Fan Engagement and Push Notification Service

## Problem/Feature Description

A sports app startup is building fan-facing features for their mobile application. They need two interconnected pieces: a fan engagement module that lets users set their favorite teams, participate in live polls during games, and send reactions, and a push notification service that sends mobile alerts to fans when key moments happen in games they care about.

The app has both iOS and Android users. Fans have complained that they get too many push notifications during game day -- every minor event triggers an alert and they end up muting the app entirely. The team wants to be smarter about which events warrant a push notification so fans stay engaged without feeling spammed.

User preferences (favorite teams) need to persist across app sessions and across devices, so a fan who sets up their favorites on their phone should see the same favorites when they open the tablet app.

## Output Specification

Produce the following files:

1. `fan-engagement.js` -- A module that handles favorite team management with persistent storage, live polls for a game, and fan reactions. Include functions for subscribing a fan to their favorite teams' channels.
2. `push-notifications.js` -- A module that registers devices for push notifications and constructs notification payloads for game events. Include logic that determines which events should trigger a push notification.
