# Live Concert Chat System

## Problem/Feature Description

A music streaming platform is adding a live chat feature for virtual concert events. During major concerts, they expect 50,000 to 100,000 concurrent viewers all wanting to chat in real-time in a single "concert chat room." The platform uses PubNub for real-time messaging.

The engineering team needs to design and implement the chat infrastructure that can handle this level of concurrency. They also need to show who is currently online and provide a way for the production team to broadcast announcements to everyone in the chat. The system should be built in JavaScript (Node.js) using the PubNub SDK.

## Output Specification

Produce the following files:

- `architecture.md`: A design document explaining the overall approach to handling 50K-100K concurrent users in a single chat room. Include the channel strategy, how messages will be distributed, how presence/online counts will work at this scale, and any steps the team needs to take with PubNub before the event. Include notes about monitoring the system during the live event.

- `chat-server.js`: A Node.js module that exports:
  - A function to initialize the PubNub connection for a given user
  - A function to determine which channel a user should join based on their user ID
  - A function to broadcast a message from the production team to all participants
  - A presence handler that works efficiently at this scale

- `chat-client.js`: A Node.js module that exports:
  - A function to connect a user to the chat (subscribing to the correct channel)
  - A function to send a chat message
  - A message listener setup
