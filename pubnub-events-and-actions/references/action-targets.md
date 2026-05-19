<!-- canonical-for: EVENTS_AND_ACTIONS_TARGETS -->
<!-- used-by: -->

# Action Targets: Webhook, SQS, Kinesis, S3, Kafka, IFTTT, AMQP

The canonical reference for each Events & Actions delivery target — what it is, when to use it, and the gotchas.

## Target Choice Quick Reference

| Target | When to use | Tier required |
|---|---|---|
| **Webhook** | Lambda, API Gateway, custom HTTP endpoint, EventBridge (via custom bridge) | Free + |
| **Amazon SQS** | Decoupled async processing in AWS | Paid |
| **Amazon Kinesis** | Streaming pipeline, real-time analytics in AWS | Paid |
| **Amazon S3** | Bulk archival, data warehouse ingestion | Paid |
| **Apache Kafka** | Streaming pipeline, microservices fanout | Paid |
| **IFTTT Webhook** | Consumer / no-code automations via IFTTT | Paid |
| **AMQP** | RabbitMQ or any AMQP 0.9.1 broker | Paid |

## Webhook

The most flexible and most used target.

### Configuration

- **Endpoint URL**: full HTTPS URL of your receiver
- **Custom headers**: arbitrary key-value headers (paid tiers); useful for shared-secret auth
- **Retries**: up to 4 retries, configurable interval (paid tiers only)

### Behavior

- HTTP `2xx` → success, no retry.
- HTTP `301` → follows up to 3 redirects.
- Other HTTP statuses → retried per policy.
- Free tier: **no retry on any failure**.

### Receiver Idempotency Is Mandatory

Webhooks can be delivered more than once (transient failure → retry). Receivers must be idempotent. Use the `id` field in each event item as a dedup key:

```javascript
const seenIds = new LRU(100_000);

app.post('/webhook/pubnub', (req, res) => {
  const events = req.body.data || [];
  for (const event of events) {
    if (seenIds.has(event.id)) continue;
    seenIds.add(event.id);
    process(event);
  }
  res.sendStatus(200);
});
```

For dedup patterns see [pubnub-reliability/references/dedup-on-merge.md](../../pubnub-reliability/references/dedup-on-merge.md).

### EventBridge via Webhook

There is no direct EventBridge target. Use the webhook target pointed at an API Gateway endpoint that calls EventBridge:

```
PubNub → Webhook → API Gateway → Lambda → EventBridge
```

Or use API Gateway's direct EventBridge integration (no Lambda).

### Lambda via Webhook

Standard pattern: API Gateway HTTP endpoint that triggers Lambda. Use API key auth via custom header.

## Amazon SQS

### Configuration

- **Queue URL**: full SQS queue URL
- **AWS credentials**: access key + secret (least-privilege IAM user with `sqs:SendMessage` only)
- **Region**

### Use When

- You want decoupled async processing of every PubNub event.
- Multiple downstream consumers can process from the same queue.
- You need at-least-once delivery into SQS.

### Standard vs FIFO

Standard SQS for high throughput, at-least-once delivery (your consumer must dedup). FIFO SQS for strict ordering at the cost of throughput.

## Amazon Kinesis

### Configuration

- **Stream name**
- **AWS credentials** (`kinesis:PutRecord` only)
- **Region**
- **Partition key strategy** (typically the [`userId`](../../pubnub-app-developer/references/sdk-patterns.md) or `channel`)

### Use When

- Real-time analytics pipeline already in Kinesis.
- You need ordering per partition key.
- Downstream consumers are Kinesis Analytics, Firehose to S3, Lambda triggers.

### Caveat: PutRecord Limits

Kinesis has per-shard write limits (1 MB/s, 1000 records/s). For high-volume PubNub keysets, ensure shard count is sized appropriately or use multiple Kinesis streams.

## Amazon S3

### Configuration

- **Bucket name**
- **Object key prefix** (E&A appends timestamp + UUID)
- **AWS credentials** (`s3:PutObject` only)
- **Region**
- **Batching settings** — almost always required for cost reasons

### Batching is Critical

S3 charges per PUT request. Without batching, every event is a separate PUT, which is expensive at scale. **Default batching: 100 items or 5 seconds, whichever first.** See [retries-and-batching.md](retries-and-batching.md) for tuning.

S3 batches over 5MB are uploaded as multipart.

### Use When

- Bulk archival of PubNub events.
- Data warehouse ingestion pipeline (S3 → Snowflake/Redshift/BigQuery).
- Compliance / audit logging.

## Apache Kafka

### Configuration

- **Bootstrap servers**: comma-separated list of broker hosts
- **Topic name**
- **Authentication**: SASL/PLAIN, SASL/SCRAM, mTLS — depends on cluster config
- **Optional schema registry settings**

### Use When

- You already have a Kafka cluster (MSK, Confluent, self-managed).
- Microservices fan-out via Kafka topics.
- Stream processing in Flink, Kafka Streams, ksqlDB.

### Network Considerations

PubNub egress must reach your Kafka brokers. For private clusters (no public bootstrap), use a public proxy or [IP allowlist PubNub's egress IPs](../../pubnub-security/references/ip-whitelisting.md).

## IFTTT Webhook

### Configuration

- **IFTTT webhook key**
- **Event name** (configured in your IFTTT applet)

### Use When

- Consumer automations (smart home, notifications via IFTTT).
- Quick prototypes with no-code action chains.

Generally not used for backend production. For backend, use plain Webhook or one of the AWS targets.

## AMQP

### Configuration

- **AMQP URI** (`amqp://user:pass@host:port/vhost`)
- **Exchange name**
- **Routing key**
- **Optional TLS**

### Use When

- You have RabbitMQ or another AMQP 0.9.1 broker.
- Existing AMQP-based integration patterns in your stack.

## Common Anti-Patterns

| Anti-pattern | Fix |
|---|---|
| Free tier in production (no retry) | Upgrade to a paid tier |
| Webhook receiver not idempotent | Dedup by event `id` |
| S3 target without batching | Configure batching (100 items / 5s default) |
| Kinesis target with too few shards | Size shards by event throughput |
| Kafka target on a private cluster with no egress path | Public proxy or IP allowlist PubNub egress |
| Hardcoded AWS credentials in the action config | Use a least-privilege IAM user (e.g., `Sender-EventsActions-Prod`, distinct from your PubNub [userId](../../pubnub-app-developer/references/sdk-patterns.md)); rotate periodically (see [key rotation](../../pubnub-keyset-management/references/key-rotation-and-hygiene.md)) |
| One action across multiple environments | One action per environment, bound to that env's keyset |

## Related Reading

- [event-types.md](event-types.md) — what's being delivered
- [filters-and-jsonpath.md](filters-and-jsonpath.md) — narrow what triggers
- [retries-and-batching.md](retries-and-batching.md) — delivery policy
