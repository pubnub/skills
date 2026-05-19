# Coalesced Cursor Publisher

## Problem/Feature Description

The collaborative cursors feature lets users see each other's cursor positions in real time. Initial implementation publishes a message on every `mousemove` event, which fires at the browser's full rate (often 60 Hz on desktop). Cost projection (from the previous scenario) shows this drives subscribe-transaction costs about 6× over the team's budget. The plan is to coalesce: queue the latest cursor position locally and publish at most once per 100ms ("latest wins").

The team also wants to distinguish coalescing-eligible streams (cursor moves) from non-coalescing-eligible streams (chat messages, order placement). The platform engineering lead wants a small, generic CoalescedPublisher class plus clear constraints on what may or may not use it.

## Output Specification

Create a JavaScript file called `coalesced-publisher.js` that exports a class `CoalescedPublisher`. The class constructor takes `(pubnub, channel, options)` where `options` may include `{ intervalMs }` (default 100). The class must:

1. Expose a method `send(state)` that stores `state` as the latest pending payload (always overwriting any previously-pending payload — last write wins, no queuing of historical states).
2. After the first `send` call, schedule a flush via `setTimeout` after `intervalMs`. Subsequent `send` calls during that window must update the pending value but must NOT schedule additional timers.
3. On flush, call `pubnub.publish({ channel, message: pending })` exactly once, then clear pending and the timer.
4. If `send` is not called during a flush window, no publish happens for that window.
5. Expose a method `flushNow()` that, if there is a pending payload, publishes it immediately and clears the timer. Safe to call even when nothing is pending.
6. Expose a method `dispose()` that clears any pending timer (does not flush).

Add a top-of-file comment that explicitly lists the documented coalescing-eligibility rules:
- Coalesce: cursor position, scroll position, game tick deltas, sensor reading where only latest matters.
- Do NOT coalesce: chat messages, order placement, any discrete event where every message matters.

Use ES module syntax.
