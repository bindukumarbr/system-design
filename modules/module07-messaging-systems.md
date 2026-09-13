# Module 7: Messaging Systems

## Core Concepts

### Synchronous vs. Asynchronous Communication

**Synchronous** (request/response, e.g. REST/gRPC over HTTP) blocks the caller until the callee responds. It's simple to reason about and gives immediate error feedback, but it couples the availability and latency of the caller to the callee — if the downstream service is slow or down, the caller stalls or fails, and a chain of synchronous calls compounds tail latency (the "fan-out amplifies p99" problem).

**Asynchronous** communication decouples producer and consumer in time and space via an intermediary (a queue or log). The producer publishes and moves on; the consumer processes on its own schedule. This buys:

- **Temporal decoupling** — consumer can be down/slow without blocking the producer.
- **Load leveling** — bursts are absorbed by the broker instead of overwhelming downstream services.
- **Multi-consumer fan-out** — one event can drive many independent side effects (notifications, analytics, fraud checks) without the producer knowing about any of them.

The cost is complexity: eventual consistency, harder debugging (no single call stack), and the need for explicit delivery-guarantee and idempotency design (covered below). The rule of thumb for system design interviews: use synchronous calls for user-facing reads that need an immediate answer (fetch a profile, check auth), and asynchronous messaging for writes/side-effects that can tolerate a short delay and that benefit from decoupling (analytics, notifications, order fulfillment, cross-service workflows).

### RabbitMQ / AMQP Concepts

RabbitMQ implements AMQP 0-9-1, a **broker-centric, smart-broker/dumb-consumer** model. The core objects:

- **Producer** publishes a message to an **exchange**, never directly to a queue.
- **Exchange** routes the message to zero or more queues based on routing rules and **bindings** (exchange → queue rules, each with a binding key).
- **Queue** is where messages actually sit until a consumer acknowledges them.
- **Consumer** subscribes to a queue and acks/nacks each message.

**Exchange types:**

- **Direct** — routes a message to the queue(s) whose binding key exactly matches the message's routing key. Used for point-to-point or simple task routing (e.g. `routing_key=email` → email queue).
- **Fanout** — ignores the routing key entirely and broadcasts to every bound queue. Used for pub/sub broadcast (e.g. an event that every downstream service must see, like cache invalidation).
- **Topic** — matches routing key against binding patterns using `*` (one word) and `#` (zero or more words), e.g. `order.*.created` matches `order.us.created`. This gives selective multicast — the AMQP equivalent of a pub/sub topic filter.
- **Headers** — routes based on message header key/value pairs instead of the routing key, using `x-match: all` or `any`. Rarely used but useful when routing criteria aren't naturally expressible as a dotted string.

RabbitMQ queues are **stateful and consumption-destructive**: once a message is acked, it's gone. This makes RabbitMQ excel at classic task-queue workloads — work items that should be processed once and discarded — with fine-grained per-message routing, priority queues, delayed messages (via plugins), and low-latency (sub-millisecond) delivery for modest throughput. It is not designed for replay or multiple independent consumer groups reading the same stream at their own pace (RabbitMQ Streams, a newer log-based add-on, partially addresses this, but classic queues do not).

### Apache Kafka Concepts

Kafka is a **distributed, append-only commit log**, not a traditional queue. Core concepts:

- **Topic** — a named stream of records, split into **partitions** for parallelism and scalability. Each partition is an ordered, immutable log; ordering is only guaranteed *within* a partition, not across the topic.
- **Offset** — a monotonically increasing per-partition sequence number identifying each record's position. Consumers track offsets (in Kafka itself, in the `__consumer_offsets` topic) to know what they've read.
- **Consumer group** — a set of consumers that cooperatively read a topic; Kafka assigns each partition to exactly one consumer within the group, so the group as a whole achieves parallelism up to the partition count, while multiple *different* consumer groups can independently read the same topic at their own pace (Kafka doesn't delete on read).
- **Log-based storage** — records are retained for a configured time/size window (not deleted on consumption), enabling replay, multiple independent readers, and stream reprocessing. Brokers replicate partitions (leader + followers) for durability; producers can require acknowledgment from all in-sync replicas (`acks=all`) for durability guarantees.
- **Broker/cluster** — Kafka scales horizontally by adding brokers and partitions; a topic's partition count is the primary lever for consumer-side parallelism, so it's typically chosen up front for expected peak throughput.

Kafka's log model makes it the natural fit for **event streaming**: high-throughput ingestion, multiple downstream consumers reading the same firehose independently (analytics, search indexing, fraud detection, notification service, all off one topic), and replay for backfills or reprocessing after a bug fix.

### Delivery Guarantee Semantics

- **At-most-once** — the message is sent/committed *before* processing; if the consumer crashes mid-processing, the message is lost, never retried. Achieved by committing the Kafka offset (or acking the RabbitMQ message) immediately on receipt, before doing the work. Simplest, but drops messages under failure — acceptable only for loss-tolerant telemetry.
- **At-least-once** — the message is retried until the consumer confirms successful processing, so a message may be *redelivered* after a crash or timeout (e.g. consumer processes it, then crashes before committing the offset/ack). Achieved by committing the offset/ack *after* processing completes successfully, not before. This is the default and most common guarantee in practice — it requires the consumer side to be **idempotent** (see below) because duplicates *will* happen.
- **Exactly-once** — the effect of processing occurs exactly once even under retries and failures. True end-to-end exactly-once across independent systems is effectively impossible in general (the "two generals" problem), but it is achievable *within* a closed system:
  - **Kafka idempotent producer** (`enable.idempotence=true`) assigns each producer a PID and sequence number per partition so the broker can deduplicate retried produce requests, guaranteeing exactly-once *writes* per partition.
  - **Kafka transactions** (`transactional.id`, read-process-write pattern) let a consumer atomically commit its output writes and its input offset commit together, giving exactly-once semantics for **Kafka-to-Kafka** stream processing (e.g. Kafka Streams).
  - For a Kafka-to-external-system sink (e.g. writing to a database or calling an API), true exactly-once still requires the sink to be idempotent or transactional itself — Kafka can only guarantee at-least-once delivery to it, and the sink must dedupe (e.g. via a unique key upsert) to achieve an exactly-once *effect*.
  - RabbitMQ has no native transactional exactly-once story across broker+consumer; it's a manual publisher-confirm + consumer-ack + idempotent-consumer combination to approximate it.

In practice, most senior-level designs target **at-least-once delivery + idempotent consumers**, because that combination is achievable with ordinary infrastructure and gives an effective exactly-once *outcome* without needing distributed transactions.

### Dead Letter Queues (DLQ)

A DLQ is a separate queue/topic that receives messages a consumer could not process successfully after a bounded number of retries (e.g. malformed payload, downstream 500s, poison messages that crash the consumer repeatedly). Purpose:

- **Prevent head-of-line blocking** — a single bad message shouldn't stall the whole partition/queue.
- **Preserve for investigation/replay** — failed messages aren't silently dropped; an on-call engineer or automated job can inspect, fix, and replay them.

Implementation patterns:
- **RabbitMQ**: native support via `x-dead-letter-exchange`/`x-dead-letter-routing-key` queue arguments — a message that's rejected (`nack`/`reject` without requeue), expires (TTL), or exceeds `x-max-length` is automatically republished to the configured DLX.
- **Kafka**: no native DLQ primitive — implemented at the application/framework level (e.g. Kafka Connect's `errors.deadletterqueue.topic.name`, or Spring Kafka's `DeadLetterPublishingRecoverer`): after N retries with backoff, the consumer publishes the failed record (plus error metadata: exception, timestamp, original topic/partition/offset) to a `topic.DLQ` topic and commits the original offset to move on.
- A good DLQ design captures enough context (original headers, retry count, stack trace/error reason, timestamp) to make replay or manual triage tractable, and has its own alerting — a growing DLQ is itself an incident signal.

### Kafka vs. RabbitMQ Decision Framework

| Dimension | Kafka | RabbitMQ |
|---|---|---|
| Model | Distributed commit log | Traditional message broker/queue |
| Throughput | Very high (100K–millions msgs/sec per cluster) | High (tens of thousands msgs/sec) |
| Message retention | Configurable time/size, replayable | Deleted after consumption/ack |
| Ordering | Per-partition | Per-queue (single consumer) |
| Routing | Topic + partition key (coarse) | Rich (direct/topic/fanout/headers, per-message routing) |
| Multiple independent consumers of same stream | Native (consumer groups) | Requires separate queues/fanout per consumer |
| Best for | Event streaming, log aggregation, activity tracking, stream processing, high-throughput pipelines that need replay | Task queues, RPC-style workflows, complex routing topologies, priority/delay queues, lower-latency small-scale messaging |
| Operational complexity | Higher (ZooKeeper/KRaft, partition rebalancing, disk management) | Lower to moderate |
| Latency | Slightly higher per-message (batching-oriented) | Very low (sub-ms) for individual messages |

Rule of thumb: choose **Kafka** when you need durable replay, multiple independent consumers over the same event stream, or very high sustained throughput (e.g. clickstream, swipe/activity events, log aggregation). Choose **RabbitMQ** when you need flexible per-message routing, request/reply semantics, priority queues, or simpler operational overhead for moderate throughput task distribution.

### Idempotency Patterns for Duplicate Message Handling

Since at-least-once delivery is the practical default, consumers must tolerate duplicates:

1. **Idempotency key / deduplication table** — every message carries a unique, producer-generated key (e.g. UUID, or a natural key like `swipe_id`). The consumer, in the same transaction as its side effect, inserts the key into a `processed_messages` table (unique constraint) or checks a fast-lookup store (Redis `SETNX` with TTL) before applying the effect; a duplicate key is a no-op.
2. **Natural idempotency via upserts** — design the write itself to be idempotent, e.g. `INSERT ... ON CONFLICT DO NOTHING/UPDATE` keyed on the business key, so replaying the same event twice converges to the same state rather than double-applying (critical for counters — use "set to" semantics or conditional increments, not blind `+1`).
3. **Idempotency key + at-most-once side effects** — for non-idempotent external calls (e.g. "send a push notification"), store an idempotency key with the *outcome* of the call (sent/failed) so a retry checks state first rather than re-invoking the external system blindly (also needed on the sender's side for APIs like Stripe/FCM that accept a client-supplied idempotency key).
4. **Exactly-once via transactional outbox / consumer offset + DB write atomicity** — commit the database write and the consumer offset (or produce-and-ack) atomically, e.g. via Kafka transactions for stream-to-stream, or a local DB transaction that writes both the business row and a "last processed offset" row for stream-to-DB.

---

## Case Study Solution: Tinder-style Matching Platform

### Problem Statement & Clarifying Requirements

**Functional requirements:**
- Users submit swipes (like/pass/superlike) on candidate profiles.
- When two users have both liked each other, the system detects a **mutual match** and creates a match record.
- Both users receive a real-time notification ("It's a match!") when a match occurs.
- Users can fetch their current list of matches.

**Non-functional requirements:**
- **High write throughput** — swipes are the dominant write path and must be ingested reliably even under bursty load (e.g. peak "prime time" traffic).
- **Low-latency match detection** — a user should see a match notification within a second or two of the second swipe, not minutes later.
- **At-least-once delivery** for the swipe → match → notification pipeline — a lost swipe event means a missed match, which is a core product failure; duplicate delivery must not create duplicate matches or duplicate notifications.
- **Durability** — swipe history and match records must survive broker/consumer crashes.
- **Horizontal scalability** — the design must scale with user growth without redesign (partitioning strategy matters).
- Out of scope for this module: the recommendation/candidate-generation algorithm (who to show), which is a separate subsystem.

### Capacity Estimation

Assume 50M daily active users, each swiping an average of 100 times/day (typical for swipe-heavy dating apps).

- Total swipes/day = 50M × 100 = 5B swipes/day.
- Average swipes/sec = 5B / 86,400 ≈ **58,000 swipes/sec**.
- Peak factor ~5x during evening "prime time" → **~290,000 swipes/sec peak**.
- Match rate: roughly 1 in 50–100 swipe pairs mutually like each other (rough industry-cited figure) → at 58K swipes/sec, assume roughly 1–2% of swipes result in a match check hit → order of a few hundred to ~1,000 matches/sec average, several thousand/sec at peak.
- Each swipe event: user_id, target_id, action, timestamp, idempotency key ≈ 150–250 bytes → at 58K/sec average that's roughly 10–15 MB/sec sustained write bandwidth, several times that at peak — well within Kafka's per-broker throughput budget with a modestly sized cluster and topic partitioned appropriately (e.g. 128–256 partitions to spread across brokers and consumer instances).
- Storage: 5B swipes/day × ~200 bytes ≈ 1 TB/day raw swipe log (before compaction/retention policy trims it, e.g. 7–30 days hot retention in Kafka, long-term in a data warehouse/S3).

### High-Level Architecture

```
 Client (mobile app)
        │  POST /swipe
        ▼
 ┌─────────────────┐
 │ Swipe Ingestion  │  (stateless, horizontally scaled API tier)
 │ Service          │
 └──────┬───────────┘
        │ produce (key = min(user_id,target_id) pair or target_id)
        ▼
 ┌─────────────────────────────┐
 │  Kafka topic: swipe-events   │  (partitioned, replicated, retained)
 └──────┬───────────────────────┘
        │ consumer group
        ▼
 ┌─────────────────┐        ┌───────────────┐
 │ Matching Engine  │──────▶│ Swipes DB      │ (append swipe, check reverse swipe)
 │ (consumer)       │        │ (Cassandra/    │
 └──────┬───────────┘        │  DynamoDB)     │
        │ on mutual match     └───────────────┘
        │ produce
        ▼
 ┌─────────────────────────────┐
 │ Kafka topic: match-events    │
 └──────┬───────────────────────┘
        │ consumer group
        ▼
 ┌──────────────────┐     ┌───────────────┐
 │ Notification      │────▶│ Push provider  │ (FCM/APNs)
 │ Service (consumer)│     │ + WebSocket    │ (for in-app real-time)
 └──────┬─────────────┘    └───────────────┘
        │ on failure after retries
        ▼
 ┌─────────────────────────────┐
 │ Kafka topic: notif-events-DLQ│
 └─────────────────────────────┘
```

- **Swipe Ingestion Service**: stateless HTTP tier, validates the request, assigns an idempotency key if the client didn't supply one, and produces to Kafka. Returns success to the client as soon as the produce is acknowledged (`acks=all`), decoupling client latency from downstream match-detection latency.
- **Kafka (`swipe-events`)** chosen as the broker for this stage: extremely high sustained throughput, natural partitioning by user pair, and replay capability (useful for backfilling the matching engine or reprocessing after a bug).
- **Matching Engine**: consumer group reading `swipe-events`; for each swipe, checks whether the *target* has already swiped-liked the *source* (reverse-lookup in the swipes store, keyed for O(1) lookup). On a mutual like, atomically writes a match record and produces a `match-events` message.
- **Notification Service**: consumer group on `match-events`; sends push notifications (FCM/APNs) and/or a WebSocket event to both users. Failures (provider timeout, invalid device token) are retried with backoff; after N failures the message goes to a DLQ topic for investigation/replay, and the primary flow moves on so one bad device token doesn't block the partition.

### API Design

```
POST /v1/swipes
Headers: Idempotency-Key: <uuid>          (client-generated, dedupes retries on flaky mobile networks)
Body: {
  "target_user_id": "u_789",
  "action": "LIKE" | "PASS" | "SUPERLIKE"
}
Response: 202 Accepted
{ "swipe_id": "sw_abc123", "status": "queued" }

GET /v1/matches?cursor=<opaque>&limit=20
Response: 200 OK
{
  "matches": [
    { "match_id": "m_555", "user_id": "u_789", "matched_at": "2026-09-05T10:00:00Z" }
  ],
  "next_cursor": "..."
}

GET /v1/matches/stream   (WebSocket / SSE upgrade for real-time "it's a match" push while app is foregrounded)
```

`POST /v1/swipes` returns `202 Accepted` rather than `200 OK` deliberately — it signals the write is queued for asynchronous processing, not synchronously finalized, which is the correct contract for an event-driven pipeline.

### Data Model

```sql
-- Swipes: append-only, partitioned by swiper for write locality
CREATE TABLE swipes (
  swipe_id        UUID PRIMARY KEY,
  swiper_id       BIGINT NOT NULL,
  target_id       BIGINT NOT NULL,
  action          SMALLINT NOT NULL,     -- 0=pass,1=like,2=superlike
  idempotency_key UUID NOT NULL,
  created_at      TIMESTAMP NOT NULL,
  UNIQUE (swiper_id, target_id, idempotency_key)  -- dedupe on retry
);
-- Secondary index / reverse-lookup table for O(1) match checks
CREATE TABLE swipe_lookup (              -- keyed for fast reverse check
  pair_key   TEXT PRIMARY KEY,           -- e.g. "min(id)_max(id)"
  a_id       BIGINT,
  b_id       BIGINT,
  a_liked    BOOLEAN,
  b_liked    BOOLEAN,
  updated_at TIMESTAMP
);

-- Matches: created once, on mutual like
CREATE TABLE matches (
  match_id        UUID PRIMARY KEY,
  user_a_id       BIGINT NOT NULL,
  user_b_id       BIGINT NOT NULL,
  matched_at      TIMESTAMP NOT NULL,
  dedupe_key      TEXT UNIQUE NOT NULL   -- "min(id)_max(id)", enforces one match per pair
);

-- Notification delivery record: idempotency + audit trail
CREATE TABLE notification_deliveries (
  delivery_id     UUID PRIMARY KEY,
  match_id        UUID NOT NULL,
  user_id         BIGINT NOT NULL,
  idempotency_key TEXT UNIQUE NOT NULL,  -- match_id + user_id, dedupes on redelivery
  status          SMALLINT NOT NULL,     -- 0=pending,1=sent,2=failed,3=dead_lettered
  attempt_count   INT NOT NULL DEFAULT 0,
  last_error      TEXT,
  updated_at      TIMESTAMP
);
```

The `swipe_lookup.pair_key` and `matches.dedupe_key` both use a canonical `min(id)_max(id)` composite so that regardless of which user's swipe arrives second, the match-detection and match-creation logic converges on the same row — this is what makes the pipeline safe under Kafka's at-least-once redelivery: a replayed swipe event just re-derives the same pair_key and hits the same row (upsert), so no duplicate match is created.

### Deep Dive

**Kafka vs. RabbitMQ for this workload.** Kafka is the right choice for `swipe-events` and `match-events`: swipe ingestion is a firehose (tens of thousands to hundreds of thousands of events/sec) that benefits from partition-level parallelism and from being replayable — if the matching engine's logic has a bug, you want to reprocess the last N hours of swipes rather than lose them, which RabbitMQ's consume-and-delete model doesn't support. Kafka also lets multiple independent consumer groups read the same `swipe-events` stream for unrelated purposes (e.g. an analytics pipeline, an anti-abuse/bot-detection pipeline, and the matching engine) without the producer or topic needing to know about all of them, whereas RabbitMQ would need explicit fanout bindings per new consumer. RabbitMQ would be reasonable for a *lower-volume, routing-heavy* piece of this system — e.g. an internal admin task queue, or if notification delivery needed complex per-provider/per-region routing rules (topic exchange keyed on `region.provider`) — but for the core swipe → match → notify pipeline, Kafka's throughput and replay properties dominate.

**Delivery guarantee and idempotency.** The pipeline targets **at-least-once delivery with idempotent consumers** end-to-end, not literal exactly-once, because the pipeline crosses Kafka → database → external push providers (FCM/APNs), and those external providers aren't part of a Kafka transaction. Concretely: the ingestion service uses an idempotent producer (`enable.idempotence=true`) so retried produces from network blips don't double-write to the log; the matching engine commits its Kafka offset only after its DB write succeeds (process-then-commit, not commit-then-process) so a crash mid-processing causes redelivery, not loss; and every write is a keyed upsert (`swipe_lookup` on `pair_key`, `matches` on `dedupe_key`) so redelivery is a safe no-op rather than a duplicate match. The notification service similarly checks `notification_deliveries.idempotency_key` (`match_id + user_id`) before calling the push provider — if a redelivered `match-events` message arrives after the first send already succeeded, the consumer sees `status=sent` and skips the call, so the user never gets a duplicate "it's a match" push.

**Dead-letter handling for notifications.** Push delivery can fail transiently (provider timeout) or permanently (stale/uninstalled device token). The notification consumer retries transient failures with exponential backoff up to a bounded attempt count, incrementing `notification_deliveries.attempt_count`; once the bound is exceeded, it publishes the original message plus failure context (error, attempt count, last provider response) to a `notif-events-DLQ` topic and marks the row `dead_lettered`, then commits the original offset so a single bad token doesn't block the rest of the partition. A separate low-volume consumer/alert watches the DLQ: permanent failures (invalid token) trigger a device-token cleanup job; transient-looking failures can be manually or automatically replayed from the DLQ after the provider incident clears. Because delivery is tracked by idempotency key, replay from the DLQ is safe even if the original attempt actually succeeded silently.

### Trade-offs and Alternatives Considered

- **Synchronous match check on swipe write** (checking the reverse swipe inline in the API request instead of via a consumer) would reduce architectural complexity and end-to-end latency for the *first* swipe of a pair, but couples the ingestion API's latency/availability to the matching datastore and doesn't scale independently — rejected in favor of async decoupling, given the 58K–290K swipes/sec range.
- **Partitioning by `swiper_id` vs. by `pair_key`**: partitioning `swipe-events` by `swiper_id` gives excellent write-side load distribution but means a mutual pair's two swipes can land on different partitions, requiring the matching engine to do a cross-partition lookup (acceptable, since the lookup goes to the shared `swipe_lookup` store, not to another partition's consumer). An alternative — partitioning by `pair_key` so both swipes of a pair always land on the same partition/consumer — simplifies match detection (in-memory, no external lookup needed) but risks hot partitions if a subset of pairs generate disproportionate traffic, and complicates key derivation before the target is known at write time in some designs; a hybrid was chosen (partition by `swiper_id`, external low-latency lookup store) as the more scalable default.
- **RabbitMQ for the whole pipeline** would simplify operations for a smaller-scale deployment but doesn't comfortably support the replay/multi-consumer-group needs at this throughput; viable only for an MVP or much lower user base.
- **Push-only vs. push + WebSocket**: relying solely on push notifications is simpler but has provider latency variance and doesn't cover the "both users have the app open right now" case well; a WebSocket/SSE channel for foregrounded users complements push for the best perceived real-time experience, at the cost of maintaining stateful connections (handled by a separate presence/connection-gateway service, out of scope here).

### How Real Systems Solve This

Tinder itself is a well-documented real-world precedent for exactly this architecture: Tinder engineers presented "Matching the Scale at Tinder with Kafka" at Kafka Summit, describing their move to a Kafka-based event pipeline to handle swipe volume and match detection at scale, reflecting the same partition-and-consumer-group pattern used above. More broadly, dating-app and high-throughput event-matching system design write-ups (e.g. system design breakdowns of Tinder-style apps) converge on the same shape: an append-only, partitioned event log for swipe ingestion (Kafka or a similar log-structured broker), a stateful matching/consumer tier backed by a low-latency key-value or wide-column store (Cassandra/DynamoDB-style, chosen for high write throughput and horizontal scalability over a single relational primary), and a downstream notification fan-out service — mirroring the general industry pattern of using Kafka as the backbone for high-volume "detect a condition across a stream of user actions, then notify" workloads.

---

## Sources

- [Matching the Scale at Tinder with Kafka — Confluent / Kafka Summit SF18](https://www.confluent.io/kafka-summit-sf18/matching-the-scale-at-tinder-with-kafka/)
- [Matching the Scale at Tinder with Kafka — SlideShare](https://www.slideshare.net/slideshow/matching-the-scale-at-tinder-with-kafka/120353076)
- [Tinder Architecture — systemdesign.one newsletter](https://newsletter.systemdesign.one/p/tinder-architecture)
- [Tinder — Fully explained System Design and Architecture — Medium](https://kasunprageethdissanayake.medium.com/tinder-fully-explained-system-design-and-architecture-1225ecdfe64e)
- [Designing Tinder: A Deep Dive into High-Level System Design — Medium](https://medium.com/@krishanu1137/designing-tinder-a-deep-dive-into-high-level-system-design-dd501c37ffe6)
- [Design Tinder: How to Design a Scalable Dating App — System Design Handbook](https://www.systemdesignhandbook.com/guides/design-tinder/)
- [Dating Application System Design — Medium (System Design Concepts)](https://medium.com/system-design-concepts/dating-application-system-design-aae411412267)
- [RabbitMQ vs Kafka vs ActiveMQ: A Complete Guide — DesignGurus](https://www.designgurus.io/blog/rabbitmq-kafka-activemq-system-design)
- [Event-Driven Architecture: Kafka vs. RabbitMQ vs. Pulsar — A 2025 Decision Framework — Java Code Geeks](https://www.javacodegeeks.com/2025/12/event-driven-architecture-kafka-vs-rabbitmq-vs-pulsar-a-2025-decision-framework.html)
- [RabbitMQ vs Kafka: Complete Messaging System Comparison — Michal Drozd](https://www.michal-drozd.com/en/guides/rabbitmq-vs-kafka/)
- [Apache Kafka vs RabbitMQ vs Amazon SQS: Message Queue Comparison — Reintech](https://reintech.io/blog/apache-kafka-vs-rabbitmq-vs-amazon-sqs-message-queue-comparison)
- [Kafka Exactly-Once: Producers + Transactions — Conduktor](https://www.conduktor.io/glossary/exactly-once-semantics-in-kafka)
- [Achieving Exactly-Once Semantics in Kafka: Producer & Consumer Idempotency — Medium](https://medium.com/@anil.goyal0057/achieving-exactly-once-semantics-in-kafka-producer-consumer-idempotency-abad50cba95c)
- [Exactly-once semantics with Kafka transactions — Strimzi](https://strimzi.io/blog/2023/05/03/kafka-transactions/)
- [RabbitMQ AMQP Concepts — RabbitMQ official documentation](https://www.rabbitmq.com/tutorials/amqp-concepts)
