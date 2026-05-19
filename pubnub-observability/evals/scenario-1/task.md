# Pre-Launch Cost Projection

## Problem/Feature Description

A product team is about to ship a new "collaborative cursors" feature in their existing PubNub-backed app. The team's finance partner has flagged that PubNub costs have been climbing, and wants every new feature to come with an explicit pre-launch transaction-count projection. The platform engineering lead wants a small JavaScript helper that takes the projected per-user behavior of the feature and the projected DAU and returns a per-category transaction estimate plus a total — and that flags categories that exceed a configured limit.

The helper will be used directly in a launch-readiness document, so the categories returned must mirror the PubNub billing taxonomy.

## Output Specification

Create a JavaScript file called `cost-projection.js` that exports a function `projectMonthlyTransactions(inputs)`. `inputs` has the shape:

```
{
  dau: number,                       // daily active users
  publishesPerUserPerSession: number,
  signalsPerUserPerSession: number,
  firesPerUserPerSession: number,
  avgFanOutPerPublish: number,       // avg subscribers per channel
  historyReadsPerUserPerSession: number,
  appContextOpsPerUserPerSession: number,
  functionInvocationsPerPublish: number,
  daysPerMonth: number,              // default 30 if not provided
  planLimits: {                      // optional per-category monthly caps
    publish?: number,
    subscribe?: number,
    signal?: number,
    fire?: number,
    history?: number,
    appContext?: number,
    functions?: number,
    total?: number,
  }
}
```

The function returns:

```
{
  categories: {
    publish: number,
    subscribe: number,
    signal: number,
    fire: number,
    history: number,
    appContext: number,
    functions: number,
  },
  total: number,
  exceeded: string[],                // names of categories (and 'total') that exceeded their planLimits entry
  dominantCategory: string,
}
```

Calculation rules:

- `publish = dau × publishesPerUserPerSession × daysPerMonth`
- `subscribe = publish × avgFanOutPerPublish` (subscribe is the fan-out multiplier of publish, NOT of signal/fire)
- `signal = dau × signalsPerUserPerSession × daysPerMonth`
- `fire = dau × firesPerUserPerSession × daysPerMonth`
- `history = dau × historyReadsPerUserPerSession × daysPerMonth`
- `appContext = dau × appContextOpsPerUserPerSession × daysPerMonth`
- `functions = publish × functionInvocationsPerPublish` (Function invocations are per publish-event, multiplied by per-publish count)
- `total = sum of all categories`
- `dominantCategory` is the key in `categories` with the largest value.
- `exceeded` is the list of category names whose value is greater than the corresponding planLimits entry; include the string `'total'` if `planLimits.total` is provided and `total > planLimits.total`.

Add a top-of-file comment explicitly stating that PubNub bills by transactions (not bytes) and that fan-out × publish rate is the dominant cost driver.

Use ES module syntax.
