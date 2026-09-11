# PubNub Vote Tallying System

This reference covers server-side vote validation, duplicate prevention, atomic tallying, fraud detection, and real-time result broadcasting using PubNub Functions and KV Store.

## Architecture Overview

Vote tallying in PubNub uses a server-side processing pipeline. Votes published to the votes channel are intercepted by a Before Publish Function that validates, deduplicates, and tallies each vote before broadcasting updated counts on the results channel.

```
Participant --> publish vote --> [Before Publish Function] --> KV Store (dedupe + tally)
                                        |                          |
                                  reject invalid              increment counter
                                        |                          |
                                        v                          v
                                   return error            publish to results channel
```

## Before-Publish vote pipeline

Use the canonical [server-authoritative Before-Publish counter pattern](../../pubnub-functions/references/functions-patterns.md#pattern-1-distributed-counter) for dedupe + atomic tally. Vote-specific behavior in this Function:

| Step | Vote domain rule |
|------|-------------------|
| Validate payload | Require `pollId`, `optionId`, `voterId` |
| Check poll open | Read `poll:<pollId>:status` from KV Store |
| Dedupe voter | Key `poll:<pollId>:voter:<voterId>` — reject `DUPLICATE_VOTE` |
| Tally | `incrCounter` on `poll:<pollId>:tally:<optionId>` and `poll:<pollId>:total` (see [functions-modules](../../pubnub-functions/references/functions-modules.md)) |
| Broadcast | Publish consolidated results to the results channel |

For atomic counter mechanics and rate-limit counters, see [Pattern 1](../../pubnub-functions/references/functions-patterns.md#pattern-1-distributed-counter) and [Pattern 7: Rate Limiting](../../pubnub-functions/references/functions-patterns.md#pattern-7-rate-limiting).

## Duplicate Vote Prevention

Duplicate prevention is enforced server-side using KV Store (see Before-Publish vote pipeline above).

| Key Pattern | Value | Purpose |
|-------------|-------|---------|
| `poll:<pollId>:voter:<voterId>` | Option ID string | Records which option a voter selected |
| `poll:<pollId>:status` | Status string | Current poll lifecycle state |
| `poll:<pollId>:tally:<optionId>` | Counter (integer) | Atomic vote count per option |
| `poll:<pollId>:total` | Counter (integer) | Total votes across all options |
| `poll:<pollId>:options` | JSON string | List of valid option IDs |

### Handling Vote Changes

If the poll allows voters to change their vote, decrement the old option and increment the new one.

```javascript
function handleVoteChange(kvstore, pollId, voterId, newOptionId) {
  const voterKey = `poll:${pollId}:voter:${voterId}`;

  return kvstore.get(voterKey).then((previousOptionId) => {
    if (previousOptionId === newOptionId) return Promise.resolve('NO_CHANGE');

    const ops = [];
    if (previousOptionId) {
      ops.push(kvstore.incrCounter(`poll:${pollId}:tally:${previousOptionId}`, -1));
    } else {
      ops.push(kvstore.incrCounter(`poll:${pollId}:total`, 1));
    }
    ops.push(kvstore.incrCounter(`poll:${pollId}:tally:${newOptionId}`, 1));
    ops.push(kvstore.set(voterKey, newOptionId));
    return Promise.all(ops);
  });
}
```

## Broadcasting Tally Updates

After each valid vote, publish `tally_update` to `poll.<pollId>.results` with consolidated counts from KV Store counters.

| Concern | Vote-specific rule |
|---------|-------------------|
| High throughput | Throttle broadcasts (~1/s) using `poll:<pollId>:lastBroadcast` in KV Store |
| Client UX | Subscribers listen on results channel only — not raw votes |

Counter reads/writes follow [Pattern 1](../../pubnub-functions/references/functions-patterns.md#pattern-1-distributed-counter) — do not re-implement the generic counter loop here.

## Vote Validation Rules

| Rule | Vote-specific check |
|------|---------------------|
| Option | `optionId` ∈ `poll:<pollId>:options` |
| Timing | Reject if past `poll:<pollId>:closesAt`; may auto-close poll |
| Multi-choice | Respect `poll:<pollId>:maxSelections` |
| Vote change | If allowed: decrement old option counter, increment new (see pipeline table above) |

## Fraud Detection Patterns

Use [Pattern 1](../../pubnub-functions/references/functions-patterns.md#pattern-1-distributed-counter) / [Pattern 7](../../pubnub-functions/references/functions-patterns.md#pattern-7-rate-limiting) for rate and fingerprint counters — do not paste generic `incrCounter` loops here.

| Pattern | Detection Method | Action |
|---------|-----------------|--------|
| Duplicate votes | KV Store voter key lookup | Reject with DUPLICATE_VOTE |
| Rapid-fire attempts | Rate counter per voter (`ratelimit:<voterId>`) | Reject after threshold |
| Session stuffing | Fingerprint counter (`poll:<pollId>:fp:<fp>`) | Reject after 3 per fingerprint |
| Late votes | Timestamp comparison | Reject and auto-close poll |
| Invalid options | Option list lookup | Reject with INVALID_OPTION |

## Initializing Poll State in KV Store

Before opening a poll, seed KV Store with the configuration the Before Publish Function needs.

```javascript
async function initializePollState(kvstore, poll) {
  const { pollId, options, settings, schedule } = poll;

  await kvstore.set(`poll:${pollId}:status`, 'created');
  await kvstore.set(`poll:${pollId}:options`, JSON.stringify(options.map(o => o.id)));
  await kvstore.set(`poll:${pollId}:maxSelections`, String(settings.maxVotesPerUser || 1));
  if (schedule.closesAt) {
    await kvstore.set(`poll:${pollId}:closesAt`, String(schedule.closesAt));
  }

  for (const option of options) {
    await kvstore.setCounter(`poll:${pollId}:tally:${option.id}`, 0);
  }
  await kvstore.setCounter(`poll:${pollId}:total`, 0);
}
```

## Error Handling

On reject, set `request.message` to `{ error, detail, timestamp }` and call `request.abort()`. See [Pattern 1](../../pubnub-functions/references/functions-patterns.md#pattern-1-distributed-counter).

### Error Codes Reference

| Error Code | HTTP Analogy | Description |
|------------|-------------|-------------|
| `INVALID_VOTE` | 400 | Missing or malformed vote fields |
| `POLL_NOT_OPEN` | 409 | Poll is not in the open state |
| `DUPLICATE_VOTE` | 409 | Voter has already voted |
| `INVALID_OPTION` | 400 | Option ID not in the poll's option list |
| `POLL_EXPIRED` | 410 | Poll close time has passed |
| `RATE_LIMITED` | 429 | Too many vote attempts |
| `SUSPICIOUS_ACTIVITY` | 403 | Fraud detection triggered |
| `TOO_MANY_SELECTIONS` | 400 | Exceeded max selections for multi-choice |
| `EMPTY_VOTE` | 400 | No options selected |

## Tally Strategies

| Strategy | Description | When to Use | Complexity |
|----------|-------------|-------------|------------|
| Simple count | Increment counter per option | Single-choice polls | Low |
| Weighted count | Multiply by voter weight | Stakeholder voting | Medium |
| Ranked aggregation | Store ranking, compute Borda/IRV offline | Elections | High |
| Running average | Maintain sum and count | Star ratings, NPS | Medium |
| Approval count | Increment for each selected option | Multi-select polls | Low |

### Weighted tally

Use `incrCounter(..., weight)` on option and total counters after duplicate-voter check — same [Pattern 1](../../pubnub-functions/references/functions-patterns.md#pattern-1-distributed-counter) pipeline with a non-1 increment.

## Best Practices

- **Always validate server-side**: Client-side validation is for UX only. The Before Publish Function is the source of truth for vote acceptance.
- **Use atomic counters for all tallies**: Use `incrCounter` via the canonical [Before-Publish counter pattern](../../pubnub-functions/references/functions-patterns.md#pattern-1-distributed-counter) — never `get` then `set` for counts.
- **Initialize KV Store before opening polls**: Ensure all option counters, status, and configuration are set before transitioning to the open state.
- **Keep KV Store keys small**: Use short, predictable key patterns. Retrieve current KV Store key/value size limits via **`how_to`** (`understand-pubnub-functions-limits-and-constraints`) or Functions docs.
- **Throttle result broadcasts for high-volume polls**: For polls expecting thousands of votes per second, broadcast on a time interval rather than after every vote.
- **Clean up KV Store after finalization**: Once results are persisted to your backend, remove KV Store keys to free up storage.
- **Log rejected votes for auditing**: Even rejected votes should be logged for post-event analysis and fraud review.
- **Test with concurrent writes**: Simulate many simultaneous votes in staging to verify your Function handles concurrency without data loss.
