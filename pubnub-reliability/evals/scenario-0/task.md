# Backoff-and-Jitter Reconnect Controller

## Problem/Feature Description

A mobile-heavy platform's incident review found two distinct reliability bugs: (1) clients reconnect on a fixed 5-second interval, so after every regional outage the platform sees a thundering herd of synchronized reconnect attempts that overload the recovering backend; (2) when a connection actually re-establishes, the retry counter never resets, so the next transient blip jumps straight to a 30-second wait.

The platform engineering lead wants a small reconnect controller that implements the documented exponential-backoff-with-full-jitter algorithm, bounds retries at a reasonable cap, resets the counter only on truly-successful (re)connects, and wires into the PubNub status listener correctly.

## Output Specification

Create a JavaScript file called `reconnect-controller.js` that exports the following:

1. `nextRetryDelay(attemptNumber, opts)` - Returns a delay in ms computed as `random in [0, min(maxMs, initialMs * multiplier^attemptNumber))`. Defaults: `initialMs = 200`, `maxMs = 30000`, `multiplier = 2`. The exponential delay is capped at `maxMs` BEFORE the jitter is applied.
2. `ReconnectController` class:
   - Constructor accepts `(opts = {})`. Stores opts on the instance.
   - Property `attempts` starts at 0.
   - Property `MAX_ATTEMPTS = 12`.
   - `scheduleReconnect(reconnectFn, onGiveUp)`:
     - If `attempts >= MAX_ATTEMPTS`, call `onGiveUp()` and return without scheduling.
     - Otherwise compute the delay via `nextRetryDelay(attempts, opts)`, increment `attempts`, and store the `setTimeout(reconnectFn, delay)` timer in a property.
   - `reset()`: zeroes `attempts` and clears any pending timer.
3. `attachStatusListener(pubnub, controller, onUserGiveUp)`:
   - Adds a `pubnub.addListener({ status })` handler that:
     - Calls `controller.reset()` AND a documented `onConnected()` hook on `PNConnectedCategory` or `PNReconnectedCategory`.
     - Calls `controller.scheduleReconnect(() => pubnub.reconnect(), onUserGiveUp)` on `PNNetworkDownCategory`, `PNNetworkIssuesCategory`, or `PNTimeoutCategory`.
   - The controller MUST NOT be reset on any other status category (e.g., a transient blip that fires `PNNetworkIssuesCategory`).

Add a top-of-file comment explicitly stating:
- Random jitter is mandatory; pure exponential backoff still synchronizes across clients.
- The attempt counter is reset only on PNConnectedCategory or PNReconnectedCategory.

Use ES module syntax.
