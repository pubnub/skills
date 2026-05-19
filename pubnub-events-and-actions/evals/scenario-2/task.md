# S3 Archival Configuration for Compliance

## Problem/Feature Description

A financial-services platform must archive every PubNub message published on its `trades.*` channels to an S3 bucket for seven years of compliance retention. The platform is on a paid PubNub tier. The cost of running 500,000 individual S3 PUTs per day for every published event would be substantial, so the data engineering lead wants the action target configured to batch aggressively, while the security team wants AWS credentials scoped down to the absolute minimum.

The platform-engineering team needs a single configuration artifact that describes:
- the E&A listener (source, event type, filter),
- the S3 action target (bucket, region, batching, credentials policy),
- the IAM permissions required,
- and a short rationale for the chosen batch sizes.

## Output Specification

Create a JSON file called `s3-archival-config.json` whose top-level shape is:

```json
{
  "listener": { ... },
  "action": { ... },
  "iam": { ... },
  "rationale": { ... }
}
```

Where:

1. `listener` includes at least: `eventSource` (one of "Messages", "Users", "Channels", "Push", "Memberships"), `eventType` (string), `filterType` (one of "none", "basic", "jsonpath"), and `filter` (the filter expression or basic-filter shape, or null).
2. `action` includes at least: `targetType` ("S3"), `bucket`, `region`, `objectKeyPrefix`, `batching` (an object with `itemCount` and `timeWindowSeconds`), and `envelopeVersion`.
3. `iam` includes at least: `actions` (an array of IAM action strings) and `principle` (a one-sentence statement of the principle being applied).
4. `rationale` includes at least: `batchingChoice` (string explaining why these batch numbers), `multipartNote` (string mentioning S3 multipart behavior at 5MB), and `costNote` (string explaining the per-PUT cost concern that motivates batching).

The `filter` should scope to channels matching `trades.*`. The batching item count and window must be chosen for high-volume archival, not for low-latency delivery.

Produce only the JSON content of the file (the file itself, valid JSON).
