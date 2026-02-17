# Secure Direct Messaging Module

## Problem/Feature Description

A healthcare startup is building a patient-provider messaging feature for their telehealth platform. Patients need to be able to have private 1:1 conversations with their assigned doctors. The platform uses PubNub for its real-time infrastructure and the team wants to use PubNub's Chat SDK for the messaging layer.

The system must support Access Manager for HIPAA compliance, meaning authentication tokens from their backend must be passed into the chat client. Each user (patient or provider) may or may not already exist in the system, so the code needs to handle both cases gracefully. The module should establish a direct conversation between two users and allow sending and receiving messages, with proper cleanup when the session ends.

## Output Specification

Create a single JavaScript file called `direct-messaging.js` that exports the following async functions:

1. `initChat(publishKey, subscribeKey, userId, authToken)` - Initializes the chat SDK and returns the chat instance.
2. `ensureUser(chat, userId, userData)` - Gets an existing user or creates one if they do not exist, and returns the user.
3. `startDirectChat(chat, otherUser)` - Creates a direct conversation channel with another user and returns the channel.
4. `connectToChannel(channel, onMessage)` - Subscribes to incoming messages on a channel, returns a cleanup function.
5. `sendMessage(channel, text)` - Sends a text message on a channel.
6. `cleanup(chat)` - Disconnects the chat instance and performs any necessary teardown.

The file should use ES module syntax (import/export).
