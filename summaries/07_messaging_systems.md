# Module 07: Messaging Systems

## Core Concepts

### Synchronous vs Asynchronous
- **Synchronous (REST/gRPC):** Caller blocks until callee responds. Good for user-facing reads (e.g., fetch profile). Bad for high-fan-out because it amplifies p99 latency (chain of slow services).
- **Asynchronous (Queues/Logs):** Decouples in time and space. Good for writes, analytics, and cross-service workflows. Provides load leveling (absorbs spikes).

### RabbitMQ (Message Broker)
- **Model:** Smart broker, dumb consumer.
- **Components:** Producer -> Exchange -> Queue -> Consumer.
- **Exchange Types:**
  - `Direct`: Exact match routing.
  - `Fanout`: Broadcasts to all queues.
  - `Topic`: Pattern matching (`order.*.created`).
- **Nature:** Consumption is destructive (once acked, message is deleted). Excellent for task queues and flexible routing. Not designed for replay.

### Apache Kafka (Event Streaming)
- **Model:** Distributed, append-only commit log.
- **Components:** Topic -> Partitions.
- **Nature:** Messages are retained (not deleted on read). Consumers track their own offsets. Allows for replay and multiple independent consumer groups reading the same stream. Excellent for massive throughput and event sourcing.

### Delivery Guarantees
- **At-most-once:** Commit offset BEFORE processing. Risk: Data loss if crash.
- **At-least-once:** Commit offset AFTER processing. Risk: Duplicates. (This is the industry standard default).
- **Exactly-once:** End-to-end exactly-once is basically a myth across external systems. It requires an idempotent consumer or a transactional outbox.

### Dead Letter Queues (DLQ)
- A queue for messages that fail repeatedly (poison pills). Prevents head-of-line blocking (where one bad message stalls the whole partition).

---

## Case Study: Tinder-style Matching Platform
- **Requirements:** 50M DAU, 100 swipes/day. At-least-once delivery for swipes so matches are not missed.
- **Estimations:** 5B swipes/day -> ~58,000 swipes/sec. Kafka handles this firehose easily.
- **Architecture:**
  - **Swipe Ingestion API:** Takes swipe, produces to Kafka `swipe-events` topic, returns 202 Accepted.
  - **Matching Engine (Consumer):** Reads `swipe-events`. Checks DB if the other user already swiped right. If mutual, writes to DB and emits `match-events`.
  - **Notification Service (Consumer):** Reads `match-events`, sends Push Notification.
- **Aha! Insights:**
  - **Idempotency Key:** Ensure mobile clients send a UUID. If they retry on bad connection, Kafka deduplicates.
  - **Canonical Pair Key:** Use `min(userA, userB)_max(userA, userB)` as the DB Primary Key for a match. This ensures that no matter who swipes first, the lock and DB insert targets the exact same row (upsert), preventing duplicate matches under retry scenarios.
