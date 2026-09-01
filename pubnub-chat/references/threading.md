# Threaded Messages

A thread is a side conversation attached to a parent message. In PubNub Chat SDK, each thread is implemented as its own channel, deterministically named from the parent message's timetoken.

## Concepts

| Concept | Definition |
|---|---|
| **Parent message** | The message a thread hangs off of |
| **Thread channel** | A child channel that holds the thread's messages |
| **Thread root** | First reply in the thread |
| **Thread preview** | Lightweight summary attached to the parent (`hasThread: true`) |

## Orchestration

| Step | Rule |
|------|------|
| Create | Use `createThread(text)` on the parent message — creates channel, sends first reply, sets `hasThread` |
| Reply | Send on the thread channel (same semantics as a regular channel) |
| Connect | Subscribe to the thread channel for live replies |
| History | Fetch thread history via the thread channel; shares parent keyset persistence settings |
| Resolve existing | `getThread()` when `hasThread: true` |
| Cleanup | `removeThread()` when explicit teardown is required |

## Constraints

- **Threads are channels** — every thread counts for billing and presence (see [usage metrics](../../pubnub-observability/references/usage-metrics.md)).
- **No nested threads** — design UI accordingly.
- **Thread channel name is deterministic** — IDs start with `PUBNUB_INTERNAL_THREAD_...`. Do not publish directly to that name.
- **Permissions** — grant Access Manager access to the thread channel separately from the parent ([access-manager](../../pubnub-security/references/access-manager.md)).
- **Cleanup** — deleting the parent message does **not** delete the thread channel.

## Common Pitfalls

| Pitfall | Mitigation |
|---|---|
| Forgot to grant access to thread channel | Re-grant on the deterministic thread channel name |
| Thread previews stale on UI | Parent exposes `hasThread`; subscribe to parent channel updates |
| Many small empty threads pollute billing | Lazy-create only on first reply via `createThread(text)` |
| Thread channels never cleaned up | Call `removeThread()` or run periodic GC of empty thread channels |
| Wanted nested threads | Not supported; flatten replies or use `quotedMessage` instead |

## Quote vs Thread

For "reply with quote" in the same channel (no side conversation), use `sendText({ quotedMessage })` — stays in the parent channel and does not create a new channel.
