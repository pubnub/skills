# Customer Support Chat Widget Setup

## Problem/Feature Description

A SaaS company is building an embeddable customer support chat widget. Their customers (businesses) will embed the widget on their websites so that end-users can chat with support agents. A key requirement is showing agents which customers are currently on the site and ready to chat, and showing end-users which agents are available.

The company has chosen PubNub for the real-time infrastructure. Each business gets its own set of chat channels (e.g., `support-acme-*`, `support-globex-*`). They need to track which users and agents are online in each channel so they can route conversations effectively.

Your task is to write a setup guide and JavaScript implementation for the presence tracking component of this widget. The guide should cover everything a developer needs to do -- from portal configuration through to the client-side code -- so that the presence system works correctly in production.

## Output Specification

Produce a single file called `presence-setup-guide.md` that contains:

1. A section on required platform/portal configuration steps (be specific about the settings)
2. A JavaScript code section that initializes the PubNub client for an agent connecting to a support channel
3. A JavaScript code section that subscribes to a channel with presence enabled
4. The heartbeat configuration recommendations for a real-time chat use case

The code should be production-quality, not just pseudocode. Use realistic channel naming like `support-{businessId}`.
