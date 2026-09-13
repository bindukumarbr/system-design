# Module 8: Event-Driven Architecture & CQRS

## Core Concepts

### Event Sourcing

Event sourcing stores the *sequence of state-changing events* as the system of record, rather than persisting only the current state. Instead of `UPDATE orders SET status='shipped' WHERE id=1`, you append `OrderShipped{orderId:1, ts:...}` to an append-only event log. Current state is a derived, rebuildable projection: `state = fold(apply, initialState, events)`.

Key properties:
- **Append-only log** — events are immutable facts; you never update or delete a past event (corrections are new compensating events, e.g. `OrderShipmentCorrected`).
- **Replay** — the current (or any past) state can be rebuilt by replaying events from the beginning, or from a **snapshot** plus subsequent events. Snapshots exist purely as a performance optimization (avoid replaying millions of events); they are never authoritative.
- **Complete audit trail** — every state transition is preserved with causality and timing, which is valuable for compliance, debugging production incidents ("what sequence of events produced this state?"), and temporal queries ("what did this account look like on March 1st?").
- **Aggregate boundary** — events are usually scoped to a DDD aggregate (e.g. `Order`, `Account`) which enforces invariants and defines the transactional consistency boundary. Each aggregate instance has its own event stream.
- **Costs** — increased complexity: you need event versioning/migration, replay tooling, snapshotting, and idiomatic query support is harder because the "current state" table doesn't exist unless a projection builds it (this is exactly why event sourcing and CQRS pair up).

When to use it: domains where audit history and temporal reasoning are first-class requirements (financial ledgers, order/inventory systems, insurance claims), or where you need to reconstruct alternate projections later without re-deriving lost information. Avoid it for simple CRUD domains where the operational overhead outweighs the benefit.

### CQRS (Command Query Responsibility Segregation)

CQRS separates the **write model** (accepts commands, enforces invariants, optimized for consistency and throughput of writes) from the **read model** (serves queries, optimized for the shapes callers actually need, often denormalized). They are two different schemas — sometimes two different databases entirely — kept in sync by an asynchronous event stream.

Why separate them:
- Read and write workloads have fundamentally different scaling and shape requirements. Writes want normalized data and strict invariant checks; reads want denormalized, pre-joined, pre-aggregated views tailored to specific UI/API shapes, and want to scale horizontally and cheaply.
- A single normalized relational schema optimized for write-safety typically requires expensive joins/aggregations to serve reads at scale (e.g. computing a leaderboard on every request). CQRS lets you materialize purpose-built read projections (e.g. Redis sorted sets, an OLAP table, an Elasticsearch index) updated asynchronously from the write side's events.
- It pairs naturally with event sourcing (the event log *is* the natural integration point to build read projections) but doesn't require it — you can do CQRS with two normal SQL databases synced via CDC or explicit events.

When NOT to use it: CQRS introduces eventual consistency between write and read models, and doubles the number of schemas/deployables to maintain. For a low-traffic CRUD service, a single model is simpler and correct. Reserve full CQRS + event sourcing for subsystems with real read/write asymmetry or an actual audit requirement — not the whole system by default.

### The Saga Pattern

A saga coordinates a business transaction that spans multiple services/databases where a single ACID transaction (2PC) isn't feasible or desirable at scale. A saga is a sequence of local transactions; each step publishes an event/command that triggers the next step. If a step fails, the saga runs **compensating transactions** to undo the effects of prior steps (there is no rollback in the ACID sense — you must design explicit "undo" operations, e.g. `RefundPayment` compensates `ChargeCard`).

Two coordination styles:

**Choreography** — each service listens for events from others and reacts, with no central coordinator. `OrderCreated` → Inventory service reserves stock and emits `StockReserved` → Payment service charges and emits `PaymentCompleted` → Shipping service ships. 
- Pros: fully decoupled, no single point of failure/bottleneck, simple for short sagas.
- Cons: the overall business process is implicit — it lives nowhere as one artifact — making it hard to reason about, monitor, and test end-to-end. Cyclic dependencies between event listeners are easy to introduce accidentally. Adding a new step means every relevant service must know new event names.

**Orchestration** — a central orchestrator (a saga/process manager) explicitly issues commands to each participant and interprets their replies, holding the saga's state machine. `OrderSagaOrchestrator` sends `ReserveStock`, waits for `StockReserved`/`StockRejected`, then sends `ChargePayment`, etc.
- Pros: the process is explicit and centrally visible/testable; a single place implements retries, timeouts, and compensation logic; easier to add new steps or complex branching.
- Cons: the orchestrator is a new component to build, deploy, and scale, and if not designed carefully can become a "smart" coupling point that participants become dependent on.

Rule of thumb: choreography suits 2–3 step sagas with low branching; orchestration wins once you have several steps, conditional branches, retries with backoff, or a need for centralized observability into "where is this transaction stuck."

### The Outbox Pattern

The **dual-write problem**: a service that must (a) update its own database and (b) publish an event/message about that update cannot do both atomically against two different systems (DB + message broker) without a distributed transaction. If the DB commit succeeds but the publish fails (or vice versa), the system silently loses events or publishes phantom events, causing consumers to diverge from the source of truth.

The **transactional outbox pattern** solves this by writing the event to an `outbox` table *in the same local ACID transaction* as the business state change. A separate relay process (or, preferably, a CDC connector like Debezium tailing the outbox table's write-ahead log) asynchronously reads new outbox rows and publishes them to the message broker (e.g. Kafka), then marks them published (or relies on log position to avoid redelivery bookkeeping). Because the business write and the outbox write are one local transaction, they succeed or fail together — the event can never be lost, and it's never published without the corresponding state change (at-least-once delivery is guaranteed; consumers must be idempotent to handle possible duplicates on relay retries). This is strictly better than a "dual write" of DB update + direct broker publish, and better than 2PC/XA, which most modern brokers and databases don't support well and which harms availability. See the AWS Prescriptive Guidance and Debezium documentation on this exact pattern (sources below).

### Event Schema Evolution

Because consumers and producers deploy independently and events are long-lived (especially in event-sourced systems where old events must remain replayable), schema changes must be backward- and forward-compatible:
- **Additive changes only** by default: add optional fields with defaults; never remove or repurpose a field meaning.
- **Explicit versioning**: embed a `schemaVersion`/`eventType` field; maintain **upcasters** that transform old event versions into the current shape at read time (common in event-sourced aggregates) rather than mutating historical data.
- **Schema registries** (Confluent Schema Registry, AWS Glue Schema Registry) enforce compatibility rules (`BACKWARD`, `FORWARD`, `FULL`) at publish time using Avro/Protobuf schemas, rejecting producer changes that would break existing consumers.
- **Weak schema coupling / tolerant reader**: consumers should ignore unknown fields and treat missing optional fields as defaults, rather than failing closed.
- For breaking changes, publish a new event type/topic version side-by-side and migrate consumers before retiring the old one (expand/contract migration).

### Change Data Capture (CDC) with Debezium

CDC captures row-level changes directly from a database's transaction/replication log and streams them as events, without requiring application code to explicitly publish anything. Debezium is the dominant open-source CDC platform: it runs as a set of Kafka Connect **source connectors** (MySQL reads the binlog, PostgreSQL reads the logical replication (WAL) stream, MongoDB reads the oplog, SQL Server reads CDC tables) and turns each committed row change into a change event (`before`/`after` image + operation type) published to a Kafka topic per source table. Debezium can also run "connector-less" via **Debezium Server** (streaming straight to Kinesis, Pub/Sub, Pulsar) or embedded as **Debezium Engine** inside a JVM app.

CDC is the cleanest way to implement the outbox-relay half of the outbox pattern: instead of a polling process querying the outbox table (which adds latency and load), Debezium tails the database log itself, picks up new outbox rows the instant they commit, and forwards them to Kafka — this is the "transactional outbox + CDC" combo recommended in Debezium's own documentation. CDC is also the standard mechanism for populating CQRS read models and for keeping search indexes/caches/data warehouses in sync with an OLTP system of record without any application-level dual writes at all.

## Case Study Solution: LeetCode-style Online Judge

### Problem Statement & Requirements

Design an online coding-judge platform (LeetCode/Codeforces/HackerRank-style): users submit code in a chosen language against a problem; the system compiles/executes it against hidden test cases in a sandbox, returns a verdict (Accepted, Wrong Answer, TLE, MLE, Runtime Error, Compile Error), and — for contests — maintains a live leaderboard ranking users by problems solved and penalty time.

**Functional requirements**
- Submit code for a problem; get a submission ID immediately.
- Judge submissions against a battery of test cases under time/memory limits; produce a verdict.
- Let the user poll/subscribe for the verdict.
- Show submission history per user.
- Maintain a real-time leaderboard during contests (rank by solved-count, then penalty/time).

**Non-functional requirements**
- Untrusted code must be fully sandboxed (arbitrary user code cannot compromise the host, other tenants, or the network).
- High burst throughput at contest start (submissions spike massively in the first minutes).
- Judging is asynchronous — the submit API must never block on execution.
- No submission may ever be silently dropped ("queued" must survive partial failures).
- Leaderboard reads must be fast (sub-100ms) and consistent enough not to visibly desync from actual solves for long.

### Capacity Estimation

Assume a contest with 100K concurrent contestants, each making ~10 submissions over a 2-hour window, with ~40% of total submissions clustering in a 10-minute burst around the start/end.

- Total submissions: 100,000 × 10 = 1,000,000
- Burst submissions: 400,000 in 600s ≈ **667 submissions/sec** peak
- Execution time per run (compile + run test cases): ~2–3s wall clock average → concurrent sandboxes needed ≈ 667 × 3 ≈ **2,000 concurrent sandbox executions**
- At ~8 sandboxes per worker VM (isolated by container/microVM, CPU-bound), that's **~250 worker machines** at peak vs. maybe 25 at steady state — a 10x autoscaling range, so workers must be an autoscaled fleet, not fixed capacity.
- Storage: 1M submissions/contest × ~2KB code+metadata ≈ 2GB/contest; trivial for a document/relational store; retained indefinitely for history.
- Leaderboard: 100K participants × small sorted-set entries — fits entirely in memory (Redis), single-digit MB.

### High-Level Architecture

```
                     ┌─────────────────┐
 Client (submit) --> │ Submission API   │---(1) validate + persist submission (status=RECEIVED)
                     │ (write path)     │---(2) write outbox row, SAME db txn
                     └────────┬─────────┘
                              │ CDC (Debezium tails outbox table WAL)
                              v
                     ┌─────────────────┐
                     │ Kafka: submissions.queued │
                     └────────┬─────────┘
                              v
                 ┌─────────────────────────┐
                 │ Judge Worker Pool         │  (autoscaled, pulls from queue)
                 │ - pull job                │
                 │ - spin sandbox (gVisor/   │
                 │   Firecracker microVM)    │
                 │ - compile + run testcases │
                 │ - emit verdict event      │
                 └────────┬──────────────────┘
                          v
                ┌────────────────────────┐
                │ Kafka: judge.completed   │
                └───────┬──────────┬──────┘
                        v          v
             ┌─────────────┐  ┌────────────────────┐
             │ Result store │  │ Leaderboard Projector│
             │ (write model,│  │ (CQRS read model     │
             │  Postgres)   │  │  builder)             │
             └──────┬───────┘  └─────────┬────────────┘
                    │                    v
                    │           ┌──────────────────┐
                    │           │ Redis sorted set   │
                    │           │ (leaderboard read)  │
                    │           └─────────┬──────────┘
                    v                     v
           Client polls/streams    Client reads leaderboard
           GET /submissions/{id}   GET /contests/{id}/leaderboard
```

### API Design

```
POST /submissions
  body: { problemId, contestId?, language, sourceCode }
  -> 202 Accepted { submissionId, status: "QUEUED" }

GET /submissions/{submissionId}
  -> 200 { submissionId, status: "QUEUED"|"JUDGING"|"ACCEPTED"|"WRONG_ANSWER"|
           "TIME_LIMIT_EXCEEDED"|"MEMORY_LIMIT_EXCEEDED"|"RUNTIME_ERROR"|"COMPILE_ERROR",
           runtimeMs?, memoryKb?, failedTestCase? }
  (Or upgrade to WebSocket/SSE: GET /submissions/{submissionId}/stream for push-based
   verdict delivery instead of polling — reduces load at contest scale.)

GET /users/{userId}/submissions?problemId=&cursor=
  -> paginated submission history (read model)

GET /contests/{contestId}/leaderboard?page=
  -> [{ rank, userId, solvedCount, penaltyMinutes }]  (served entirely from Redis ZSET)
```

### Data Model

**Write side (source of truth, normalized, e.g. PostgreSQL)**
```
submissions(
  id UUID PK, user_id, problem_id, contest_id NULLABLE,
  language, source_code, status, created_at, updated_at
)
judge_results(
  submission_id FK, verdict, runtime_ms, memory_kb,
  failed_test_case_id NULLABLE, judged_at
)
outbox(
  id UUID PK, aggregate_id (=submission_id), event_type,
  payload JSONB, created_at, published BOOLEAN  -- or omitted if CDC uses log position
)
```

**Read side (CQRS projections)**
```
Redis ZSET  leaderboard:{contestId}
  member = userId
  score  = (solvedCount << 40) | (MAX_PENALTY - penaltyMinutes)   -- packs both ranking
                                                                      criteria into one
                                                                      sortable score
submission_history (denormalized doc store, e.g. per user, per problem, latest verdict)
```

### Deep Dive: CQRS Split, Outbox, and Saga Style

**CQRS split.** The write path is optimized purely for correctness and durability under burst load: `POST /submissions` does one thing — validate, persist to `submissions` (status=RECEIVED) and write the corresponding outbox row, in a single local transaction — then returns 202 immediately. It never touches the sandbox, never computes a leaderboard, never blocks on judging. The read path is a completely separate set of models: the leaderboard is a Redis ZSET rebuilt incrementally by a dedicated **Leaderboard Projector** consumer that reacts to `judge.completed` events and does an atomic `ZADD`/`ZINCRBY`; submission history is a denormalized store optimized for "show me this user's last 50 submissions" without joining `submissions` and `judge_results` at read time. Because the read models are just projections of the event stream, they can be rebuilt from scratch at any time by replaying `judge.completed` from Kafka — which is exactly the event-sourcing replay property applied pragmatically to just the read side rather than the whole system.

**Outbox guarantee.** The critical failure mode to avoid: the submission API commits `submissions` row as RECEIVED, then crashes (or the broker is unreachable) before publishing `SubmissionQueued` — the user's code silently never gets judged. The outbox pattern eliminates this: the outbox row is written in the *same Postgres transaction* as the submission insert, so it is impossible to have one without the other. A Debezium connector tails Postgres's WAL, picks up the new outbox row the instant it commits, and republishes it to the `submissions.queued` Kafka topic — no separate polling process, no window where the event could be lost, and no second network call that could fail independently of the DB commit. Judge workers consume that topic idempotently (keyed by submissionId, with a status check before executing) so any at-least-once redelivery from the relay is a safe no-op rather than a duplicate judge run.

**Choreography vs. orchestration for the judging saga.** The saga here is: *submit → execute → score → update leaderboard*. This case favors a **lightweight orchestrated/state-machine style over the worker, with choreography for the read-side fan-out**: the judge worker itself acts as a mini-orchestrator for the sandbox-execution steps (it explicitly drives compile → run each test case → aggregate verdict → emit one terminal `judge.completed` event; a state machine like this benefits from centralized retry/timeout logic per submission, and the operator needs one place to see "this submission is stuck in COMPILING"). But once `judge.completed` is emitted, everything downstream — leaderboard update, history projection, notification/webhook to the client, analytics — is pure **choreography**: independent consumers of the same event, with no coordinator, because these are fan-out side effects with no compensating logic and no ordering dependency between them. This hybrid is the practical answer: use orchestration where a multi-step process needs centralized state/retry/compensation (the judging pipeline itself), and choreography where independent services simply need to react to a fact that already happened (post-judgment fan-out) — full choreography across the whole submit-to-leaderboard chain would make "why hasn't this submission been judged in 30s" nearly untraceable, while full orchestration of every downstream reader would create unnecessary coupling to a coordinator that has no compensating action to run anyway.

### Trade-offs and Alternatives Considered

- **Polling vs push for verdict delivery**: polling is simpler and stateless but wastes API capacity at contest scale (thousands of clients polling every second); WebSocket/SSE push scales better for spiky contest traffic but adds connection-management complexity. A hybrid (short-poll fallback, push where supported) is common in production.
- **Synchronous judging** (execute inline in the request) was rejected: it couples API latency to sandbox cold-start and execution time, and a burst of submissions would directly saturate the API tier instead of being buffered by a queue — the async queue + worker pool is what lets 667 req/s of *submissions* be decoupled from the ~2,000 concurrently-running sandboxes needed to serve them.
- **Full event sourcing for submissions** (storing every judge event forever as the sole source of truth, deriving all state by replay) was considered but rejected as overkill: submissions are append-mostly with a short-lived status lifecycle and no requirement to replay/rewind business state — CQRS's read/write split delivers the scaling benefit needed without event-sourcing's added replay/versioning machinery. Event sourcing does earn its keep, however, in the outbox/event log itself and in the leaderboard projector, which is literally rebuilt by replaying `judge.completed`.
- **CDC-based outbox relay (Debezium) vs. application-level polling relay**: Debezium adds an operational dependency (Kafka Connect) but removes polling latency and load; a polling relay is simpler to run but adds seconds of latency and DB read pressure at high submission rates. At contest scale, CDC wins.
- **Sandbox isolation choice**: plain Docker containers are cheaper to start but share the host kernel (higher escape risk for arbitrary contestant code); gVisor or Firecracker microVMs add a stronger isolation boundary at modest startup-latency cost. Given untrusted code is the entire threat model here, the stronger isolation is worth the latency.

### How Real Systems Solve This

- **Judge0** (open-source, widely embedded in judge platforms and AI code-execution products) isolates each run in a cgroup/namespace-restricted sandbox with strict CPU/memory/process/file-size limits and a queue-backed worker model, matching the async submit → queue → sandbox → verdict shape described above.
- Public system-design write-ups of LeetCode/Codeforces-style platforms converge on the same core shape: an API tier that only enqueues, a horizontally-autoscaled sandboxed worker fleet as the bottleneck resource, and a cache/sorted-set-backed leaderboard rather than a live aggregation query, precisely because recomputing rankings from raw submission rows on every leaderboard read cannot meet contest-time latency and load requirements.
- The transactional-outbox + CDC combination described above (rather than ad hoc dual writes to a DB and a broker) is the pattern independently documented by AWS's own prescriptive architecture guidance and by the Debezium project itself as the standard fix for the dual-write problem in exactly this kind of "persist, then notify a queue" flow.

## Sources

- [Debezium Architecture — Debezium Documentation](https://debezium.io/documentation/reference/stable/architecture.html)
- [Debezium project homepage](https://debezium.io/)
- [Transactional outbox pattern — AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html)
- [The Outbox Pattern: Reliable Event Publishing for Microservices — Streamkap](https://streamkap.com/resources-and-guides/outbox-pattern-explained)
- [Outbox Pattern for Reliable Event Publishing — Conduktor](https://www.conduktor.io/glossary/outbox-pattern-for-reliable-event-publishing)
- [What is Debezium? Understanding Change Data Capture — Streamkap](https://streamkap.com/resources-and-guides/what-is-debezium-en)
- [judge0/judge0 — sandboxed online code execution system (GitHub)](https://github.com/judge0/judge0)
- [Online Judge / LeetCode System Design — systemdesignschool.io](https://systemdesignschool.io/problems/leetcode/solution)
- [Designing LeetCode: Online Code Judge System — DEV Community](https://dev.to/matt_frank_usa/designing-leetcode-online-code-judge-system-4372)
- [System Design — LeetCode-style online Judge — Puneet Patwari (Medium)](https://medium.com/@patwaripuneet15/system-design-leetcode-style-online-judge-3375a2d2e8b9)
- [Design a Global-Scale Online Judge System — Dilip Kumar (Medium)](https://dilipkumar.medium.com/design-an-online-judge-like-leetcode-30ff9e73b248)
