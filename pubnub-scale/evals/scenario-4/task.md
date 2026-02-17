# Secure Mobile Messaging App

## Problem/Feature Description

A healthcare startup is building a mobile messaging app (using a web-based mobile framework) for doctors and nurses to communicate about patients. The app runs on phones and tablets that are carried around hospital wards all day. Battery life is a critical concern -- staff complain their devices die before shifts end. The app subscribes to multiple channels: a critical "code blue" emergency channel, a team channel, several patient-specific discussion channels, and a general announcements channel.

Messages contain sensitive patient health information and must be encrypted end-to-end. When a staff member locks their phone or switches to another app, the messaging app should minimize battery usage while still receiving critical alerts. When they return to the app, they should see any messages they missed.

The team needs a PubNub-based implementation in JavaScript that handles all of these concerns.

## Output Specification

Produce the following files:

- `pubnub-config.js`: A Node.js module that exports a function to create and return a properly configured PubNub instance for this use case. Include comments explaining the configuration choices.

- `mobile-chat.js`: A Node.js module that exports:
  - A function to initialize the chat for a user, loading recent history and then subscribing for new messages
  - A function that handles the app going to background (e.g., triggered by a visibility change event), adjusting subscriptions appropriately -- the critical emergency channel must remain subscribed
  - A function that handles the app returning to foreground, restoring full subscriptions and catching up on missed messages
  - A message listener that handles incoming messages efficiently

- `design-notes.md`: A brief document explaining the design tradeoffs made for battery life, which PubNub configuration parameters were adjusted and why, and how encryption is handled for stored messages.
