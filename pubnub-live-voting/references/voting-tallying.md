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

## Before Publish Function for Vote Validation

This function intercepts every vote before it reaches subscribers. It validates the vote, checks for duplicates, increments the tally, and publishes updated results.

```javascript
// PubNub Function: Before Publish on poll.*.votes
export default (request) => {
  const kvstore = require('kvstore');
  const pubnub = require('pubnub');
  const { pollId, optionId, voterId } = request.message;

  if (!pollId || !optionId || !voterId) {
    request.message = { error: 'INVALID_VOTE', detail: 'Missing required fields' };
    return request.abort();
  }

  return kvstore.get(`poll:${pollId}:status`).then((status) => {
    if (status !== 'open') {
      request.message = { error: 'POLL_NOT_OPEN', detail: `Poll is ${status || 'unknown'}` };
      return request.abort();
    }

    const voterKey = `poll:${pollId}:voter:${voterId}`;
    return kvstore.get(voterKey).then((existingVote) => {
      if (existingVote) {
        request.message = { error: 'DUPLICATE_VOTE', detail: 'Already voted' };
        return request.abort();
      }

      return kvstore.set(voterKey, optionId)
        .then(() => kvstore.incrCounter(`poll:${pollId}:tally:${optionId}`, 1))
        .then(() => kvstore.incrCounter(`poll:${pollId}:total`, 1))
        .then(() => broadcastTally(pubnub, kvstore, pollId))
        .then(() => request.ok());
    });
  });
};
```

## Duplicate Vote Prevention

Duplicate prevention is enforced server-side using KV Store. Each voter's choice is stored with a key combining poll ID and voter ID.

### KV Store Key Schema

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

## Atomic Counters with incrCounter

The `kvstore.incrCounter()` method provides atomic increment/decrement operations, preventing race conditions when thousands of votes arrive simultaneously.

```javascript
kvstore.incrCounter('poll:abc:tally:opt-a', 1);    // Increment by 1
kvstore.incrCounter('poll:abc:tally:opt-a', -1);   // Decrement by 1
kvstore.getCounter('poll:abc:tally:opt-a');          // Read current value
```

| Operation | Method | Atomicity | Notes |
|-----------|--------|-----------|-------|
| Increment | `incrCounter(key, n)` | Atomic | Safe for concurrent access |
| Decrement | `incrCounter(key, -n)` | Atomic | Counter can go below zero |
| Read | `getCounter(key)` | Eventually consistent | May lag slightly behind writes |
| Reset | `setCounter(key, 0)` | Atomic | Use when resetting polls |

## Broadcasting Tally Updates

After each valid vote, the Function fetches all option tallies and publishes a consolidated result.

```javascript
function broadcastTally(pubnub, kvstore, pollId) {
  return kvstore.get(`poll:${pollId}:options`).then((optionsJson) => {
    const options = JSON.parse(optionsJson || '[]');

    const counterPromises = options.map((optId) =>
      kvstore.getCounter(`poll:${pollId}:tally:${optId}`).then((count) =>
        ({ optionId: optId, count: count || 0 })
      )
    );

    return Promise.all(counterPromises).then((tallies) =>
      kvstore.getCounter(`poll:${pollId}:total`).then((totalVotes) => {
        const counts = {};
        tallies.forEach((t) => { counts[t.optionId] = t.count; });

        return pubnub.publish({
          channel: `poll.${pollId}.results`,
          message: {
            type: 'tally_update', pollId,
            counts, totalVotes: totalVotes || 0, updatedAt: Date.now()
          }
        });
      })
    );
  });
}
```

### Throttled Broadcasting

For high-throughput polls, broadcast at most once per second to avoid overwhelming clients.

```javascript
function throttledBroadcast(pubnub, kvstore, pollId) {
  const throttleKey = `poll:${pollId}:lastBroadcast`;
  return kvstore.get(throttleKey).then((lastTime) => {
    const now = Date.now();
    if (lastTime && (now - parseInt(lastTime, 10)) < 1000) return Promise.resolve();
    return kvstore.set(throttleKey, now.toString())
      .then(() => broadcastTally(pubnub, kvstore, pollId));
  });
}
```

## Vote Validation Rules

### Option Validation

```javascript
function validateOption(kvstore, pollId, optionId) {
  return kvstore.get(`poll:${pollId}:options`).then((optionsJson) => {
    const validOptions = JSON.parse(optionsJson || '[]');
    if (!validOptions.includes(optionId)) {
      throw new Error(`INVALID_OPTION: ${optionId} is not valid for poll ${pollId}`);
    }
    return true;
  });
}
```

### Time-Based Validation

```javascript
function validatePollTiming(kvstore, pollId) {
  return kvstore.get(`poll:${pollId}:closesAt`).then((closesAt) => {
    if (closesAt && Date.now() > parseInt(closesAt, 10)) {
      return kvstore.set(`poll:${pollId}:status`, 'closed').then(() => {
        throw new Error('POLL_EXPIRED: Poll has passed its close time');
      });
    }
    return true;
  });
}
```

### Multiple Choice Validation

```javascript
function validateMultipleChoice(kvstore, pollId, selectedOptions) {
  return kvstore.get(`poll:${pollId}:maxSelections`).then((maxStr) => {
    const max = parseInt(maxStr, 10) || 1;
    if (!Array.isArray(selectedOptions)) throw new Error('INVALID_FORMAT: Must be an array');
    if (selectedOptions.length > max) throw new Error(`TOO_MANY_SELECTIONS: Max ${max}`);
    if (selectedOptions.length === 0) throw new Error('EMPTY_VOTE: Select at least one');
    return true;
  });
}
```

## Fraud Detection Patterns

### Rate Limiting per Voter

```javascript
function checkRateLimit(kvstore, voterId) {
  const rateKey = `ratelimit:${voterId}`;
  return kvstore.getCounter(rateKey).then((attempts) => {
    if (attempts && attempts > 10) throw new Error('RATE_LIMITED: Too many attempts');
    return kvstore.incrCounter(rateKey, 1);
  });
}
```

### Session Fingerprint Detection

```javascript
function checkSessionFingerprint(kvstore, pollId, fingerprint) {
  const fpKey = `poll:${pollId}:fp:${fingerprint}`;
  return kvstore.getCounter(fpKey).then((count) => {
    if (count && count >= 3) throw new Error('SUSPICIOUS_ACTIVITY: Multiple votes from session');
    return kvstore.incrCounter(fpKey, 1);
  });
}
```

### Fraud Detection Summary

| Pattern | Detection Method | Action |
|---------|-----------------|--------|
| Duplicate votes | KV Store voter key lookup | Reject with DUPLICATE_VOTE |
| Rapid-fire attempts | Rate counter per voter | Reject after threshold |
| Session stuffing | Fingerprint counter | Reject after 3 per fingerprint |
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

### Centralized Error Handler

```javascript
function handleVoteError(request, errorCode, detail) {
  request.message = { error: errorCode, detail, timestamp: Date.now() };
  return request.abort();
}
```

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

### Weighted Tally Example

```javascript
function handleWeightedVote(kvstore, pollId, optionId, voterId, weight) {
  const voterKey = `poll:${pollId}:voter:${voterId}`;
  return kvstore.get(voterKey).then((existing) => {
    if (existing) throw new Error('DUPLICATE_VOTE');
    return kvstore.set(voterKey, optionId);
  }).then(() => kvstore.incrCounter(`poll:${pollId}:tally:${optionId}`, weight))
    .then(() => kvstore.incrCounter(`poll:${pollId}:total`, weight));
}
```

## Best Practices

- **Always validate server-side**: Client-side validation is for UX only. The Before Publish Function is the source of truth for vote acceptance.
- **Use atomic counters for all tallies**: Never use `get` then `set` to update counts. The `incrCounter` method prevents race conditions under concurrent load.
- **Initialize KV Store before opening polls**: Ensure all option counters, status, and configuration are set before transitioning to the open state.
- **Keep KV Store keys small**: Use short, predictable key patterns. KV Store has per-key size limits of 256 bytes for keys and 32 KB for values.
- **Throttle result broadcasts for high-volume polls**: For polls expecting thousands of votes per second, broadcast on a time interval rather than after every vote.
- **Clean up KV Store after finalization**: Once results are persisted to your backend, remove KV Store keys to free up storage.
- **Log rejected votes for auditing**: Even rejected votes should be logged for post-event analysis and fraud review.
- **Test with concurrent writes**: Simulate many simultaneous votes in staging to verify your Function handles concurrency without data loss.
