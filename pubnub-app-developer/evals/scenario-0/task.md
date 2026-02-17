# Customer Session Manager

## Problem/Feature Description

A SaaS company is building a browser-based customer support dashboard where support agents connect to real-time channels to receive incoming customer tickets. Each support agent logs in with their company credentials and needs to maintain a persistent connection to PubNub throughout their shift. The engineering team has noticed that every time an agent refreshes their browser tab, they appear as a new user in the PubNub presence system, which inflates their monthly active user (MAU) counts and causes confusion in presence tracking.

The team needs a JavaScript module that properly initializes the PubNub SDK for browser-based support agents. The module should handle the case where agents may reload their page multiple times during a shift but should always be recognized as the same user. The module should also expose a method to connect to a given support queue channel and begin receiving messages.

## Output Specification

Produce a single JavaScript file named `support-agent-client.js` that exports:

1. An initialization function that sets up the PubNub client for a given agent (accepting the agent's authenticated user ID and PubNub keys as parameters)
2. A function to connect to a support queue channel and listen for incoming messages
3. A function to send a reply message on a channel

The code should be production-ready for a browser environment.
