# Multi-Intent Question Requires Clarification

## Problem/Feature Description

A founder building an on-demand grocery delivery service writes in:

> "OK so we're going all-in on PubNub. I need to set up our dev/staging/prod keysets, then build the customer-facing real-time order tracking page, AND we need to know when delivery drivers go offline mid-route. Where should I start with PubNub? I'm just looking for direction across all of this."

The user has named three clearly distinct PubNub concerns in one breath: environment setup, order tracking, and presence/offline detection. Each routes to a different specialist skill with different MCP tools. The router's job is to recognize the multi-intent nature of the question and ask which to address first, rather than guessing or chaining three handoffs.

## Output Specification

Do NOT produce a handoff block. Instead, do all of the following:

1. Acknowledge that the user has described multiple distinct intents.
2. Briefly list the distinct intents you identified (one short bullet or sentence each is fine).
3. Ask the user which one to address first.
4. Do not name a single specific specialist skill or MCP tool to start with; do not chain multiple handoffs.
5. Do not include any implementation code.
