# Chat Message Enrichment Function

## Problem/Feature Description

Our team is building a real-time chat application using PubNub. Messages published to chat channels need to be enriched before they reach subscribers. Specifically, when a user sends a message, the function should look up the sender's cached profile from persistent storage, attach it to the message, and increment a per-channel message counter so we can track activity. If the sender profile is missing, the function should still allow the message through with a default "Anonymous" author name.

The function should run on any channel under the `chat` namespace (e.g., `chat.general`, `chat.support`). Messages arrive with `userId` and `text` fields and need to be augmented with the looked-up profile and the updated channel message count before delivery.

## Output Specification

Produce the following files:

1. **`enrich-message.js`** -- The complete PubNub Function source code.
2. **`deployment.md`** -- A short document explaining how to deploy this function, including what trigger type to use, what channel pattern to configure, and any configuration steps needed (e.g., secrets, vault entries). Also mention what to do before going live on production channels.
