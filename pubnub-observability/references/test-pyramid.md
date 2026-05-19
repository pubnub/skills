<!-- canonical-for: TEST_PYRAMID -->
<!-- used-by: -->

# Test Pyramid for Real-Time Apps

The canonical reference for testing PubNub features at the right level: unit tests for pure logic, integration tests for round-trips, load tests for fan-out and concurrency.

## Layered Test Strategy

```
              ┌──────────────────┐
              │   End-to-end     │   Few; slow; on real devices
              ├──────────────────┤
              │     Load         │   Few; pre-launch & pre-large-event
              ├──────────────────┤
              │   Integration    │   Medium; CI on a test keyset
              └──────────────────┘
              │     Unit         │   Many; fast; no network
              └──────────────────┘
```

## Unit Tests (Many, Fast, No Network)

Test pure logic that doesn't need PubNub:

| Subject | What to assert |
|---|---|
| Envelope construction | `schema_version`, `message_id`, `sent_at` present and correct |
| Reducer / state-update | Given a sequence of messages, reducer produces expected state |
| [Schema-version handler dispatch](../../pubnub-reliability/references/schema-versioning.md) | Each version routes to the correct handler; unknown drops gracefully |
| [Dedup logic](../../pubnub-reliability/references/dedup-on-merge.md) | Set/LRU correctly identifies duplicate `message_id` |
| Filter / transform logic that runs on every message | Pure-function output for a controlled input |
| [Backoff calculation](../../pubnub-reliability/references/backoff-and-jitter.md) | Sequence of delays is correct; jitter is in expected range |
| [Queue management](../../pubnub-reliability/references/queue-and-retry.md) | enqueue / dequeue ordering, capacity handling, dead-letter movement |

Mock PubNub completely:

```javascript
const mockPubnub = {
  publish: jest.fn().mockResolvedValue({ timetoken: '17000000000000000' }),
  subscribe: jest.fn(),
  addListener: jest.fn(),
};

test('publishWithLogging emits message_id in publish call', async () => {
  await publishWithLogging(mockPubnub, 'test-channel', { message_id: 'abc' });
  expect(mockPubnub.publish).toHaveBeenCalledWith({
    channel: 'test-channel',
    message: expect.objectContaining({ message_id: 'abc' }),
  });
});
```

## Integration Tests (Medium, CI, Test Keyset)

Test full round-trips against a real but isolated PubNub keyset.

### Setup

- Use a dedicated test keyset (separate from dev — see [environment separation](../../pubnub-keyset-management/references/keysets-and-environments.md)).
- Use unique channel names per test (`test-${uuid()}`; see [naming](../../pubnub-app-developer/references/sdk-patterns.md)) so parallel test runs don't collide.
- Cap test runtime — most receive tests should fail fast at 5–10s.

### What to Assert

| Subject | Assertion |
|---|---|
| Publish-receive round trip | Message published is received with same `message_id` and matching payload |
| Subscribe + reconnect | After simulated disconnect, subscription resumes with no missed messages (within short-term cache) |
| [History fetch](../../pubnub-history/references/pagination-and-ordering.md) | After publish, the message appears in `fetchMessages` |
| [App Context CRUD](../../pubnub-app-context/references/users.md) | Set + get round-trip with custom fields |
| [Access Manager](../../pubnub-security/references/access-manager.md) | A token with limited grants permits expected ops and rejects others |
| [Function](../../pubnub-functions/references/functions-basics.md) execution | A `before publish` Function transforms the message as expected |

### Pattern

```javascript
test('publish-receive round trip preserves envelope', async () => {
  const channel = `test-${crypto.randomUUID()}`;
  const envelope = {
    schema_version: 1,
    message_id: crypto.randomUUID(),
    sent_at: Date.now(),
    payload: { hello: 'world' }
  };

  const received = new Promise(resolve => {
    pubnub.addListener({ message: e => resolve(e) });
    pubnub.subscribe({ channels: [channel] });
  });

  await new Promise(r => setTimeout(r, 500));   // give subscribe time to settle
  await pubnub.publish({ channel, message: envelope });

  const e = await Promise.race([received, timeoutAfter(5000)]);
  expect(e.message.message_id).toBe(envelope.message_id);
  expect(e.message.payload).toEqual({ hello: 'world' });
});
```

For [`addListener` and `pubnub.subscribe` mechanics](../../pubnub-app-developer/references/publish-subscribe.md) see the canonical owner.

## Load Tests (Few, Pre-Launch, Test Keyset)

Run before launch and before any [large event](../../pubnub-scale/references/large-events.md) (10k+ concurrent subscribers).

### What to Measure

| Metric | Target |
|---|---|
| Sustained publish rate per connection | Per your SDK's documented limit |
| Subscribe fan-out latency at N concurrent subscribers | p50 < 100ms, p99 < 500ms (typical) |
| Reconnect storm: thousands of clients reconnecting at once | All reconnect within bounded backoff window without errors |
| History fetch concurrency | N concurrent `fetchMessages` calls succeed without throttling |
| [Presence event](../../pubnub-presence/references/presence-events.md) volume | `join` / `leave` events scale with N |

### Test Setup

- **Use a test keyset.** Loading prod can trip [DDoS protection](../../pubnub-security/references/dos-mitigation.md).
- **Coordinate with PubNub Support** before any test exceeding ~10k concurrent connections — they may pre-allocate capacity.
- **Use jittered ramp** (not all clients connecting at t=0) to mimic real traffic.
- **Capture both client-side and server-side metrics**: pull [`get_pubnub_usage_metrics`](usage-metrics.md) before and after.

### Pattern

```javascript
async function loadTest(N, channel) {
  const clients = [];
  for (let i = 0; i < N; i++) {
    // see [`new PubNub(...)` config options](../../pubnub-app-developer/references/sdk-patterns.md)
    const c = new PubNub({ subscribeKey, userId: `loadtest-${i}` });
    c.addListener({ message: () => receivedCount++ });
    c.subscribe({ channels: [channel] });
    clients.push(c);
    await sleep(jitter(50));   // ramp over time
  }

  await sleep(5000);
  const t0 = Date.now();
  await publisher.publish({ channel, message: { ts: t0 } });

  await sleep(5000);
  const elapsedMs = Date.now() - t0;
  console.log(`Fan-out to ${N}: ${receivedCount}/${N} received in ${elapsedMs}ms`);

  for (const c of clients) c.unsubscribeAll();
}
```

## End-to-End Tests (Few, Real Devices, Staging)

For mobile and complex web flows, run a small E2E suite on real devices in a staging environment that mirrors prod configuration:

- App cold start → connect → receive
- Background → foreground transition → reconnect
- Network interruption → reconnect → catch-up
- Long offline → queue drain on reconnect
- App update flow with [schema version](../../pubnub-reliability/references/schema-versioning.md) bump

E2E tests are slow and brittle. Keep the suite small. Most logic should be covered by unit + integration.

## What Not to Test

| Anti-test | Why skip |
|---|---|
| PubNub itself (does `publish` work?) | Trust the platform; test your code |
| Network reliability of PubNub | Use [reliability patterns](../../pubnub-reliability/SKILL.md), don't try to test the unreliable parts |
| Demo keyset behavior | [Demo keys are public and throttled](../../pubnub-keyset-management/references/demo-keys.md); they're not your prod keys |

## Anti-Patterns

| Anti-pattern | Fix |
|---|---|
| Integration tests against prod keyset | Use a dedicated test keyset |
| Shared channel name across parallel test runs | Use a unique channel per test (`test-${uuid()}`) |
| Long-running tests that hang on missed messages | Use `Promise.race` with a timeout |
| Loading prod to "see what happens" | Pre-coordinate with PubNub Support; use a test keyset |
| Mocking PubNub at integration layer | Mock at unit, real PubNub at integration |
| No load test before a known traffic spike (Super Bowl, product launch) | Always run one beforehand |

## Related Reading

- [logging-correlation.md](logging-correlation.md) — what to capture during tests
- [cost-and-payload-hygiene.md](cost-and-payload-hygiene.md) — load tests verify cost projections
- [usage-metrics.md](usage-metrics.md) — pre/post test metrics
- [pubnub-scale/references/large-events.md](../../pubnub-scale/references/large-events.md) — coordinate with Support
