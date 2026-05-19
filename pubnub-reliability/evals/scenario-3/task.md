# Persistent Offline Publish Queue

## Problem/Feature Description

A delivery-driver mobile app loses connectivity in tunnels and parking garages mid-shift. The product owner wants drivers' status updates ("arrived at pickup", "started route") to never be lost when the network drops — they must be queued locally on the device and published the moment the device is online again. The previous in-memory queue lost everything when the app was killed by the OS. The platform engineering lead wants a persistent queue with bounded depth, a dead-letter store for items that exhaust retries, and the drain wired to the documented status-event triggers.

The team uses an injected `db` adapter with the methods `put(table, item)`, `get(table, id)`, `getAll(table)`, `delete(table, id)`, `count(table)`. The queue tables are `'outbox'` and `'dead-letter'`. The adapter is assumed to be backed by IndexedDB or SQLite — the queue itself does not implement persistence.

## Output Specification

Create a JavaScript file called `offline-queue.js` that exports the following:

1. A constant `MAX_QUEUE_DEPTH = 1000` and a constant `MAX_ATTEMPTS = 5`.
2. `enqueue(db, channel, message)` - Async. Counts the outbox via `db.count('outbox')`. If the depth is already `>= MAX_QUEUE_DEPTH`, returns `{ ok: false, reason: 'queue-full' }` without writing. Otherwise puts an item with shape `{ id: crypto.randomUUID(), enqueued_at: Date.now(), channel, message, attempts: 0 }` into `'outbox'` and returns `{ ok: true, id }`. The `message` argument is expected to already include its own `message_id` from the producer's createMessage step.
3. `drainOnce(pubnub, db, isOnline)` - Async. If `isOnline()` returns false, returns immediately without throwing. Otherwise reads all `outbox` items via `db.getAll('outbox')` (assumed FIFO by enqueue order) and for each item:
   - Calls `pubnub.publish({ channel: item.channel, message: item.message })` once.
   - On success, deletes the item from `outbox`.
   - On failure, increments `item.attempts`. If the new `attempts` is `> MAX_ATTEMPTS`, deletes from `outbox` AND puts to `dead-letter`. Otherwise re-puts to `outbox` with the incremented `attempts`.
   - Does NOT throw out of drainOnce; logs failures and continues.
4. `attachDrainTriggers(pubnub, db, isOnline)` - Wires the drain:
   - Add a status listener that calls `drainOnce` on `PNReconnectedCategory` or `PNConnectedCategory`.
   - `setInterval(() => drainOnce(pubnub, db, isOnline), 30000)` as a 30-second backstop.
5. `inspectDeadLetter(db)` - Async. Returns the dead-letter items mapped to `{ enqueued_at, age_minutes, attempts, channel, message_id: item.message.message_id }` — never auto-discards.

Add a top-of-file comment explicitly stating:
- The queue is persistent via the injected db adapter (NOT in-memory only).
- Drain triggers include status events (PNReconnectedCategory / PNConnectedCategory) AND a 30s periodic backstop.
- Dead-letter items are surfaced via inspectDeadLetter and never auto-discarded.
- Each enqueued message carries its own message_id from the producer so retries are safe.

Use ES module syntax.
