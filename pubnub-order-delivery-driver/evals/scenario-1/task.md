# Order Status Management System

## Problem/Feature Description

A logistics company is migrating their delivery order tracking from a REST-polling architecture to real-time updates using PubNub. Their current system has a recurring bug where orders sometimes jump from "placed" directly to "en-route" due to race conditions in their backend, confusing customers and breaking their analytics pipeline. They need a robust order status management system that enforces valid transitions and keeps all stakeholders (customers, drivers, dispatchers) synchronized.

The team needs a JavaScript implementation of the order lifecycle manager that handles publishing status changes, validates that each transition is legal, and ensures the dispatch team's dashboard receives all updates. They also want a PubNub Function that acts as a server-side gatekeeper to block invalid transitions even if a buggy client tries to push them. The system must handle the full delivery lifecycle from order placement through completion, including failure and reassignment scenarios.

## Output Specification

Produce the following files:

1. `order-status-manager.js` -- The client-side status publisher class that validates transitions and publishes updates to the appropriate channels.
2. `status-validation-function.js` -- A PubNub Function that validates order status transitions server-side, preventing invalid state changes even from buggy clients.
3. `order-status-manager.test.js` -- Tests that verify the state machine allows valid transitions, rejects invalid ones, and handles failure/reassignment paths correctly.
