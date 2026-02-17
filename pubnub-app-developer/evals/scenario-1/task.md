# Private Messaging Backend Module

## Problem/Feature Description

A dating app startup is building a private messaging feature where any two matched users can have a 1:1 conversation. When two users match, the app opens a private chat between them. The team has encountered a bug where the same two users sometimes end up with two different chat channels depending on who initiated the conversation first, leading to split message histories.

The team needs a JavaScript module that handles the channel management and messaging logic for 1:1 private chats. Given any two user IDs, the module must always resolve to the same channel name so messages are never split. The module should also handle sending and receiving messages on these private channels.

## Output Specification

Produce a single JavaScript file named `direct-messaging.js` that exports:

1. A function that takes two user IDs and returns the channel name for their private conversation
2. A function to initialize the PubNub client and start listening for messages on a given direct message channel
3. A function to send a message on a direct message channel, accepting the message text and sender information

The code should include appropriate channel naming conventions and demonstrate how a client would connect and exchange messages.
