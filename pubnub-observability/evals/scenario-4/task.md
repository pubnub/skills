# Weekly Usage-Metrics Review

## Problem/Feature Description

A platform team has been surprised by a 35% jump in their PubNub bill in the last billing cycle. The finance team agreed that a weekly review of usage metrics — per-environment — could have caught the issue earlier. The team's platform engineer wants a small JavaScript module that, given two weeks of usage-metric data and the team's DAU for each week, produces a structured anomaly report broken down by transaction category, with cost-per-DAU computed for both weeks.

The module will be run from the team's analytics tooling, which already has the data in memory; this scenario is only about the analysis logic, not about pulling from the MCP tool.

## Output Specification

Create a JavaScript file called `usage-review.js` that exports a function `weeklyReview(input)`. `input` has the shape:

```
{
  keysetId: string,                  // for the report header — e.g. "acme-chat-prod"
  current: {
    weekStart: string,               // ISO date
    categories: {
      publish: number,
      subscribe: number,
      signal: number,
      fire: number,
      history: number,
      appContext: number,
      functions: number,
      presence: number
    },
    dau: number
  },
  previous: { ... same shape as current ... },
  thresholds: {
    categorySlackMultiplier: number, // e.g. 1.5 — category > 1.5x previous → slack
    categoryPageMultiplier: number,  // e.g. 3 — category > 3x previous → page
    totalSlackMultiplier: number,    // e.g. 1.5 — total > 1.5x previous → slack
    totalPageMultiplier: number      // e.g. 2 — total > 2x previous → page
  }
}
```

The function returns:

```
{
  keysetId,
  current: { ...input.current, total: <sum>, costPerDau: <total / dau> },
  previous: { ...input.previous, total: <sum>, costPerDau: <total / dau> },
  totalRatio: <current.total / previous.total>,
  costPerDauRatio: <current.costPerDau / previous.costPerDau>,
  categoryRatios: { publish: ratio, subscribe: ratio, ... },
  dominantCategoryCurrent: <category name with max value in current>,
  alerts: [
    { level: 'slack' | 'page', category: 'subscribe' | 'total' | ..., ratio: number }
  ]
}
```

Alert rules:

- If a category's ratio (current/previous) is strictly greater than `categoryPageMultiplier`, push `{ level: 'page', category, ratio }`.
- Otherwise if it's strictly greater than `categorySlackMultiplier`, push `{ level: 'slack', category, ratio }`.
- Same applies to `total` (push with `category: 'total'`).
- Page-level alerts take precedence over slack-level for the same category — never push both.
- Categories with ratio ≤ slackMultiplier are not alerted at all.

Add a top-of-file comment stating:
- The review is per-keyset (per-environment); mixing environments distorts attribution.
- Cost-per-DAU normalization is what reveals per-user efficiency changes, since flat cost-per-DAU with rising DAU is usually fine.

Use ES module syntax.
