# Price Alert Notification System

## Problem/Feature Description

A trading platform wants to let users set custom price alerts on stocks. When a stock hits a user's target price (either rising above or falling below a threshold), the user should receive an instant notification. The platform already streams live stock quotes through PubNub, so the alert evaluation should happen server-side as quotes flow through, minimizing latency.

The system has two parts: a client-side alert manager that lets users create and delete alerts and listen for triggered notifications, and a server-side PubNub Function that intercepts quote messages, evaluates them against stored alert conditions, and fires notifications to the right users. Alerts should support multiple condition types beyond simple price thresholds, including percentage change and volume spikes. Once an alert fires, it should not fire again for the same condition until reset.

Build both the client-side alert management module and the server-side PubNub Function.

## Output Specification

Produce the following files:

- `alert-manager.js` -- A client-side JavaScript module that:
  - Allows users to create alerts with various condition types (price above, price below, percentage change, volume)
  - Allows users to delete existing alerts
  - Subscribes to and listens for triggered alert notifications for the user

- `alert-function.js` -- A PubNub Function (server-side) that:
  - Evaluates incoming quote messages against stored alert conditions
  - Supports multiple alert types
  - Sends notifications to the appropriate user when conditions are met
  - Prevents duplicate notifications for the same alert

- `package.json` -- Dependencies needed

No additional files are required.
