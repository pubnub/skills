# Help a Developer Replace Demo Keys with Their Own Keyset

## Problem/Feature Description

A junior engineer copied a "Hello, PubNub!" sample from the PubNub website into their company's prototype service. They've added some custom channels and started sharing the prototype with internal stakeholders. A staff engineer reviewing the prototype is concerned: the sample they copied uses PubNub's public demo keys, which are publicly documented, throttled, and offer no isolation between users. The junior engineer asks for a clear write-up they can paste into their pull-request that explains why this is unsafe and how to migrate to their company's own dev keyset.

The staff engineer wants a single Markdown file they can attach to the PR review.

## Output Specification

Create a Markdown file called `demo-keys-warning.md` with these exact section headings (in this order):

```markdown
# Why Demo Keys Are Not Safe Here

## What Demo Keys Are

## Why This Is a Problem for Our Prototype

## What to Use Instead

## Migration Steps
```

Specific content requirements:

- The "What Demo Keys Are" section must state that demo keys are publicly documented and that anyone using the same keys on the same channel can see the messages.
- The "Why This Is a Problem for Our Prototype" section must list at least three of: no isolation, throttled / aggressive rate limits, no add-on guarantees, no security (anyone can publish or subscribe), no SLA, no support.
- The "What to Use Instead" section must say to create a free PubNub account and use the company's own keyset, and must note that the free tier is appropriate for development.
- The "Migration Steps" section must be a numbered list with at least four steps that include:
  1. Sign up for / locate the company's PubNub account at pubnub.com.
  2. Create an app (or use the existing app) and let it auto-generate a keyset.
  3. Configure add-ons per environment.
  4. Store the new keys in environment variables (or a secrets manager) — never in source control.

Do not include any actual literal PubNub demo keys (`pub-c-` / `sub-c-` strings) in the document — the goal is to discourage their use, not to copy them around.
