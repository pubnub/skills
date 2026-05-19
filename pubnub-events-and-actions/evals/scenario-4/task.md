# Split a Mixed Request Between Functions and E&A

## Problem/Feature Description

A product manager hands an architect three new requirements for an existing PubNub-backed chat app:

> 1. "Every chat message containing a known piece of personally identifiable information (PII) should be redacted before any subscriber sees it — replace the PII with `***`."
> 2. "We need to forward every chat message that lands in `support.priority.*` channels to our existing Slack workflow so support agents see them in Slack too."
> 3. "Once a day, we want a dump of every message published across the platform to land in our S3-based data lake so the data team can do batch analysis."

The architect must decide which requirements belong in PubNub Functions (per-message transform/validate/enrich) and which belong in Events & Actions (no-code routing to external systems). The decision document will be reviewed by the platform engineering manager.

## Output Specification

Create a Markdown file called `routing-decision.md` with the following structure (use these exact section headings):

```markdown
# Routing Decision: Functions vs Events & Actions

## Requirement 1: PII Redaction
- Owner: ...
- Reason: ...
- Sketch: ...

## Requirement 2: Slack Forwarding for Priority Support
- Owner: ...
- Reason: ...
- Sketch: ...

## Requirement 3: Daily S3 Data Lake Dump
- Owner: ...
- Reason: ...
- Sketch: ...

## Cross-cutting Notes
- ...
```

Each `Owner` line is a single value: `pubnub-functions` or `pubnub-events-and-actions`.

Each `Reason` is one or two short sentences explaining the choice in terms of the E&A-vs-Functions distinction (E&A routes, Functions transforms in flight).

Each `Sketch` is a short bulleted list of the high-level shape of the implementation (e.g., what event source / what action target / what trigger type / what filter would be used). No full code is required; this is a routing decision document, not an implementation.

The `Cross-cutting Notes` section must mention at least:
- that PubNub Free tier has no retry and would be inappropriate for the production requirements above,
- that the S3 target requires batching to keep costs reasonable.
