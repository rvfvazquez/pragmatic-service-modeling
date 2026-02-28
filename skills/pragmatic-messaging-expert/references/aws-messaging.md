# AWS Messaging Reference: SQS, SNS, EventBridge, Kinesis, MSK

## Table of Contents
1. [Amazon SQS](#amazon-sqs)
2. [Amazon SNS](#amazon-sns)
3. [Amazon EventBridge](#amazon-eventbridge)
4. [Amazon Kinesis](#amazon-kinesis)
5. [Amazon MSK (Managed Kafka)](#amazon-msk)
6. [Decision Matrix](#decision-matrix)
7. [Combination Patterns](#combination-patterns)

---

## Amazon SQS

**Type**: Managed message queue — point-to-point
**Delivery model**: At-least-once delivery (Standard) or exactly-once + FIFO ordering (FIFO queue)
**Consumer model**: Competing consumers — each message is processed by one consumer
**Retention**: 1 minute to 14 days (default 4 days)
**Throughput**: Standard: unlimited. FIFO: 300 msg/s per API call (up to 3,000 with batching)
**Visibility timeout**: Message hidden from other consumers while being processed
**DLQ**: Dead Letter Queue for failed messages after N processing attempts
**Message size**: Up to 256KB (larger payloads via S3 + SQS Extended Client)
**Ordering**: Standard: best-effort ordering. FIFO: strict FIFO per message group

### When SQS wins
- Decoupling two services: producer doesn't need to know about consumers
- Background job processing: tasks that can be retried and are idempotent
- Rate limiting: consumers process at their own pace, absorbing traffic spikes
- Simple point-to-point command delivery (do X with these parameters)
- Reliable delivery with automatic retry and DLQ for failures
- Lambda trigger: SQS → Lambda is a native, battle-tested integration

### When SQS struggles
- Fan-out to multiple independent consumers (each message goes to only one consumer — use SNS for fan-out)
- Real-time streaming with replay and multi-consumer reads (use Kinesis)
- Complex event routing by content/type (use EventBridge)
- Very high-throughput FIFO ordering (FIFO queues have throughput limits)

### Key design considerations
- **FIFO vs Standard**: Use Standard for maximum throughput and when ordering is not critical. Use FIFO when order within a message group is mandatory (e.g., order events for a single entity).
- **Visibility timeout**: Set to at least 2x the maximum expected processing time to avoid duplicate delivery.
- **DLQ**: Always configure a DLQ. Set `maxReceiveCount` to define retry attempts before DLQ.
- **Long polling**: Use `WaitTimeSeconds=20` to reduce empty receive calls and cost.
- **Message group ID (FIFO)**: Partition ordering by entity (e.g., orderId) to maximize parallelism while maintaining per-entity ordering.

---

## Amazon SNS

**Type**: Managed pub/sub messaging — fan-out
**Delivery model**: At-least-once delivery; push-based to subscribers
**Consumer model**: Multiple independent subscribers each receive a copy of the message
**Subscribers**: SQS, Lambda, HTTP/S endpoints, email, SMS, mobile push (APNs, FCM)
**Retention**: No retention — fire-and-forget (if subscriber is unavailable, message is lost unless backed by SQS)
**Throughput**: Very high (millions of messages/second)
**Filtering**: Message filtering per subscription (reduce noise — subscribers receive only relevant messages)
**FIFO topics**: SNS FIFO + SQS FIFO for ordered fan-out (limited throughput, like FIFO SQS)

### When SNS wins
- Fan-out: one event needs to notify multiple independent services simultaneously
- Notifications: email alerts, SMS, mobile push, HTTP webhooks
- Decoupling N-to-M communication patterns
- Combining with SQS: SNS fan-out → multiple SQS queues (buffering per consumer)
- Cross-account and cross-region event broadcasting

### When SNS struggles
- Single consumer scenarios (SQS alone is simpler and cheaper)
- Retention and replay (no message history — if a consumer misses a message, it's gone unless backed by SQS)
- Complex routing logic based on event content (EventBridge has richer filtering)
- High-volume streaming analytics (use Kinesis)

### Key design considerations
- **SNS + SQS fan-out pattern**: This is the most common combination on AWS. SNS fans out to multiple SQS queues. Each SQS queue provides buffering, retry, and DLQ for each consumer independently.
- **Message filtering**: Use subscription filter policies to route only relevant events per subscriber. Reduces unnecessary Lambda invocations and SQS messages.
- **FIFO SNS**: Use SNS FIFO when you need fan-out + strict ordering. Pair with SQS FIFO queues.
- **Message attributes**: Add metadata (event type, source, etc.) to enable effective subscription filtering.

---

## Amazon EventBridge

**Type**: Serverless event bus — event routing and integration platform
**Delivery model**: At-least-once delivery; rule-based routing to targets
**Consumer model**: Rules match events and route to one or more targets (Lambda, SQS, SNS, Step Functions, API Gateway, Kinesis, etc.)
**Retention**: Events on the bus are not retained (use SQS/Kinesis as target for durability)
**Throughput**: Default 10,000 events/sec per bus (adjustable)
**Filtering**: Rich content-based filtering on any event field (exact match, prefix, suffix, numeric range, exists/not-exists)
**Schema Registry**: Auto-discovers and documents event schemas — generates SDKs
**SaaS Integration**: 35+ SaaS event sources (Datadog, Zendesk, Shopify, GitHub, etc.)
**Event Archive**: Archive and replay historical events
**Pipes**: EventBridge Pipes for point-to-point integration with filtering and enrichment

### When EventBridge wins
- Event-driven architecture with complex routing: route by event type, source, or content
- AWS service events: react to EC2 state changes, CodePipeline stages, S3 events, CloudTrail API calls, etc.
- SaaS integration: receive events from third-party services without writing polling code
- Decoupling: producers emit events to the bus; consumers add rules independently (producers don't know about consumers)
- Scheduled rules: cron-based event generation (replacing CloudWatch Events)
- Cross-account event routing via event bus policies
- Schema-driven development: auto-generated event schemas and code bindings

### When EventBridge struggles
- Very high throughput streaming (>10k/sec continuous — use Kinesis)
- Message ordering guarantees (EventBridge does not guarantee strict ordering)
- Long message retention or replay of large volumes (archive is available but limited vs. Kinesis)
- Point-to-point commands with retry and DLQ (SQS is simpler for this)
- Complex stream processing (aggregations, windows, joins — Kinesis/MSK with Flink)

### Key design considerations
- **Custom vs default bus**: Use the default bus for AWS service events. Create custom event buses for application events — better isolation and access control.
- **Dead letter queues for rules**: Configure a DLQ on EventBridge rules to capture failed deliveries.
- **Event schema**: Design a consistent event schema (source, detail-type, detail) across all producers — invest in the Schema Registry from day one.
- **EventBridge Pipes**: Simplifies point-to-point integrations (e.g., SQS → enrichment Lambda → target) without custom code.
- **Cross-account**: Share custom buses across accounts for centralized event routing in multi-account AWS Organizations.

---

## Amazon Kinesis

### Kinesis Data Streams

**Type**: Real-time data streaming — ordered, multi-consumer
**Delivery model**: At-least-once; pull-based (consumers poll shards)
**Consumer model**: Multiple independent consumer groups each read all records (unlike SQS)
**Retention**: 24 hours (default) up to 365 days (extended retention, additional cost)
**Ordering**: Strict ordering per shard (partition key determines shard placement)
**Throughput per shard**: 1 MB/s write, 2 MB/s read; scale by adding shards
**Replay**: Yes — consumers can rewind and re-read from any point within retention window
**Consumer types**: Standard (polling, 2 MB/s shared per shard) or Enhanced Fan-Out (dedicated 2 MB/s per consumer — push-based)

### Kinesis Data Firehose (Delivery Streams)

**Type**: Managed ETL + delivery pipeline (not a streaming platform)
**Use case**: Ingest data → transform (Lambda) → deliver to S3, Redshift, OpenSearch, Splunk, HTTP endpoint
**No consumer code**: Fully managed, no consumer to write
**Latency**: Near-real-time (60-second buffering minimum, or 1MB buffer)
**Transformation**: Inline Lambda for format conversion (JSON → Parquet, enrichment)

### When Kinesis wins
- Real-time data pipelines: clickstream, log aggregation, IoT telemetry, application metrics
- Multiple independent consumers need to read the same stream (unlike SQS)
- Replay capability needed: new consumer bootstraps from T-24h, or replay after bug fix
- Ordered processing per entity (partition by entity key to ensure ordering on same shard)
- High-volume ingestion at sustained throughput (>10K events/sec)
- Analytics: feed Kinesis Data Analytics (Apache Flink) for real-time aggregations and windowing

### When Kinesis struggles
- Complex stream processing and joins across streams → MSK with Kafka Streams or Flink is more powerful
- Fan-out for notification/alerting → SNS or EventBridge are simpler
- Simple point-to-point job queue → SQS is simpler and cheaper
- Kafka ecosystem tooling required (Connect, Kafka Streams, Schema Registry) → use MSK

### Key design considerations
- **Shard count**: Each shard = 1 MB/s write, 2 MB/s read. Plan capacity and use On-Demand mode for automatic scaling.
- **Partition key design**: Distribute writes evenly across shards. Hot partitions cause throttling.
- **Enhanced Fan-Out**: Use when multiple consumers need high throughput per consumer (dedicated 2 MB/s pipe per consumer).
- **Checkpointing**: Consumers track position via sequence numbers. Use KCL (Kinesis Client Library) or Lambda for checkpointing.
- **Kinesis + Firehose**: Use Kinesis Data Streams as the input to Firehose when you need both real-time consumers AND archival to S3.

---

## Amazon MSK (Managed Streaming for Apache Kafka)

**Type**: Fully managed Apache Kafka — enterprise streaming platform
**Delivery model**: At-least-once (default) or exactly-once (with Kafka transactions + idempotent producers)
**Consumer model**: Consumer groups — each group reads all partitions; multiple groups read independently
**Retention**: Configurable (hours to unlimited with tiered storage)
**Ordering**: Strict ordering per partition (partition key determines assignment)
**Throughput**: Scales with brokers and partitions — enterprise-grade (millions of msg/sec)
**Ecosystem**: Full Kafka API compatibility: Kafka Connect, Kafka Streams, KSQL, Schema Registry
**Deployment**: Provisioned (broker count and size) or Serverless MSK (auto-scaling, pay per use)
**Connectivity**: VPC-based; supports IAM auth, SASL/SCRAM, TLS

### When MSK wins
- Enterprise with existing Apache Kafka expertise and investment
- Kafka ecosystem required: Kafka Connect for source/sink connectors, Kafka Streams or Apache Flink for complex stream processing
- Hybrid cloud or on-premises integration (Kafka is the de facto standard for event streaming across environments)
- Complex stream processing: joins, aggregations with time windows, exactly-once semantics
- Very large-scale streaming (high partition counts, long retention with tiered storage)
- Regulatory/compliance requirement to own the streaming platform
- Schema Registry for strong schema governance across producers and consumers

### When MSK struggles
- Team has no Kafka experience — operational complexity is significant even with managed service
- Simple AWS-native event routing (EventBridge is far simpler)
- Greenfield startup without Kafka expertise — Kinesis Data Streams covers most use cases with less overhead
- Cost-sensitive small workloads — MSK provisioned has fixed broker costs; consider MSK Serverless

### Key design considerations
- **MSK Serverless vs Provisioned**: Serverless for variable/unknown workloads with no capacity planning. Provisioned for predictable, high-throughput workloads where cost optimization matters.
- **Partition count**: More partitions = more parallelism but more overhead. Balance based on consumer count and throughput.
- **Consumer groups**: Each microservice should have its own consumer group to read independently.
- **Kafka Connect**: Use for ingesting from/to databases, S3, OpenSearch, etc. — avoid writing custom producers/consumers for standard integrations.
- **MSK Connect**: AWS-managed Kafka Connect — runs connectors without managing Kafka Connect workers.
- **Tiered storage**: Enable for long retention at lower cost (hot data on broker, cold data on S3).

---

## Decision Matrix

| Criterion | SQS | SNS | EventBridge | Kinesis | MSK |
|-----------|-----|-----|-------------|---------|-----|
| Pattern | Point-to-point queue | Fan-out pub/sub | Event routing bus | Data stream | Kafka platform |
| Consumers | One per message | All subscribers | Matched rule targets | Multiple groups | Multiple groups |
| Message retention | Up to 14 days | None (fire-forget) | None (archive optional) | 1–365 days | Configurable |
| Replay | No | No | Limited (archive) | Yes (within retention) | Yes (within retention) |
| Ordering | FIFO option | No (FIFO topic option) | No | Per shard | Per partition |
| Throughput | Unlimited (Standard) | Very high | ~10K/sec default | MB/s per shard | Millions/sec |
| Routing logic | None (single destination) | Subscription filters | Rich content-based rules | None (consumers decide) | None (consumers decide) |
| SaaS integration | No | No | 35+ native sources | No | Via Kafka Connect |
| Exactly-once | No (FIFO: at-least-once) | No | No | No (Enhanced: yes with KCL) | Yes (with transactions) |
| Kafka ecosystem | No | No | No | No | Yes (full compatibility) |
| Operational complexity | Low | Low | Low | Medium | Medium-High |
| Team expertise needed | Low | Low | Low | Medium | High (Kafka) |
| Cost model | Pay per request | Pay per request | Pay per event | Pay per shard-hour | Pay per broker/hour or serverless |

---

## Combination Patterns

### Pattern 1: SNS + SQS (Fan-out with buffering)
**The classic AWS pattern**. SNS fans out to multiple SQS queues. Each queue provides independent buffering, retry, DLQ, and rate-limiting per consumer.
**Use case**: Order placed event → SNS topic → SQS queue for inventory service, SQS queue for email service, SQS queue for analytics service. Each processes independently at their own pace.

### Pattern 2: EventBridge + SQS/Lambda (Event-driven routing)
**Use case**: Route events by type/source to different handlers. Complex rules determine which Lambda or SQS queue receives each event.
**Example**: CodePipeline events → EventBridge → Lambda for Slack notification, SQS for audit logging, Step Functions for post-deploy testing.

### Pattern 3: Kinesis Data Streams + Lambda (Real-time processing)
**Use case**: High-volume streaming data with multiple consumers. Lambda processes records in real-time; other consumers (Firehose → S3, Kinesis Analytics) read the same stream.
**Example**: Clickstream → Kinesis → Lambda (real-time personalization) + Firehose (S3 for analytics) + Kinesis Analytics (live dashboard metrics).

### Pattern 4: MSK (Kafka) + Kafka Connect (Hybrid integration)
**Use case**: Enterprise streaming platform connecting on-premises systems, AWS services, and external data stores.
**Example**: On-premises Oracle DB → Kafka Connect (Debezium CDC) → MSK → Kafka Streams processing → Kafka Connect Sink → S3 + OpenSearch.

### Pattern 5: EventBridge + Kinesis (Event bus + streaming)
**Use case**: EventBridge routes high-level application events; Kinesis handles high-volume data streams. EventBridge rules can target Kinesis streams.
**Example**: User action events → EventBridge (routing by event type) → Kinesis (high-volume clickstream for analytics).

### Anti-patterns to avoid
- **SQS for fan-out**: Each message goes to ONE consumer. Use SNS → multiple SQS queues for fan-out.
- **SNS without SQS buffer**: If consumers can be slow or offline, SNS alone loses messages. Always add SQS as a buffer.
- **MSK for simple AWS-native queuing**: Massive operational overhead vs SQS/EventBridge for simple use cases.
- **Kinesis for low-volume simple queuing**: SQS is cheaper and simpler when you don't need replay or multi-consumer streaming.
- **EventBridge for high-volume streaming** (>10K/sec continuous): It's an event router, not a streaming platform. Use Kinesis or MSK.
- **Ignoring DLQ**: Any SQS-based or EventBridge-based integration without a DLQ loses messages silently on failure. Always configure DLQs.
