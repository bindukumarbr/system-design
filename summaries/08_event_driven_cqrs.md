# Module 08: Event-Driven Architecture & CQRS

## Core Concepts

### Event Sourcing
- Instead of storing current state, store the sequence of state-changing events in an append-only log.
- **Pros:** Perfect audit trail, time-travel debugging, ability to replay events to build new views.
- **Cons:** High complexity. Only use for financial ledgers, critical auditing, or strict CQRS systems.

### CQRS (Command Query Responsibility Segregation)
- Separates the **Write Model** (normalized, strict validation) from the **Read Model** (denormalized, optimized for queries).
- **Why?** Reads and writes scale differently. A complex leaderboard is too expensive to compute on the fly (Write Model). CQRS lets you async-update a Redis ZSET (Read Model) that handles massive read traffic instantly.

### The Saga Pattern
- Used for distributed transactions across multiple microservices without 2PC (Two-Phase Commit).
- **Choreography:** Decentralized. Services listen to events and react. Good for 2-3 step workflows.
- **Orchestration:** Centralized coordinator explicitly commands services. Good for complex branching and retries.
- **Compensation:** Since there are no ACID rollbacks, failures must trigger explicit "undo" events (e.g., `RefundPayment`).

### Transactional Outbox Pattern & CDC (Critical Pattern!)
- **The Dual-Write Problem:** Saving an order to Postgres AND publishing an event to Kafka is NOT atomic. One can fail.
- **The Outbox Fix:** Save the order and write an event to an `outbox` table in the *same Postgres ACID transaction*.
- **Change Data Capture (CDC):** Tools like **Debezium** tail the Postgres transaction log (WAL) and automatically stream the outbox row into Kafka. Guaranteed at-least-once delivery with zero dual-write risk.

---

## Case Study: LeetCode-style Online Judge
- **Requirements:** 100k concurrent contest users, submit code, execute in sandbox, live leaderboard.
- **Estimations:** 667 submissions/sec peak. 2000 concurrent sandboxes running (takes 3s each).
- **Architecture:**
  - **Submission API:** Takes code, saves to DB + `outbox` table in one transaction. Returns 202 Accepted.
  - **Debezium CDC:** Streams outbox into Kafka `submissions.queued`.
  - **Judge Workers:** Pull from Kafka, spin up sandboxes (Firecracker/gVisor), emit `judge.completed` to Kafka.
  - **Leaderboard Projector (CQRS):** Reads `judge.completed`, updates a Redis ZSET.
- **Aha! Insights:**
  - The Write API does NOTHING but save the code to the DB. It doesn't touch the execution sandbox or the leaderboard. This allows it to absorb massive spikes.
  - The judging pipeline acts as an Orchestrator, but the downstream updates (leaderboard, notifications) are pure Choreography.
