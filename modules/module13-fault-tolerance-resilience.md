# Module 13: Fault Tolerance & Resilience

## Core Concepts

### High Availability via Redundancy and Failover

High availability (HA) is the property of a system that keeps serving traffic despite the failure of individual components. It is achieved almost entirely through **redundancy** (no single instance of anything is a single point of failure) plus **failover** (automatic detection of a failed component and rerouting to a healthy one).

- **Redundancy patterns**: active-active (all replicas serve traffic, load is spread and any node's loss just reduces capacity) vs active-passive (a standby replica is idle or replicates data until promoted). Active-active gives better resource utilization and faster failover (no promotion delay) but requires the workload to tolerate multiple concurrent writers or a partitioning scheme.
- **Levels of redundancy**: process-level (multiple app instances behind a load balancer), AZ-level (multi-AZ deployment within a region so a data-center failure doesn't take down the service), and region-level (multi-region for surviving a regional outage, at the cost of cross-region replication lag and much higher operational complexity).
- **Failover mechanics**: health checks (liveness/readiness probes) feed a load balancer or service registry; when a node fails checks it is removed from rotation. For stateful systems, failover means promoting a replica (e.g., a database standby) and requires consensus (quorum) to avoid split-brain — two nodes both believing they are primary.
- **The math of composed availability**: for a request path with N components in series, each independently available at 99.9%, the overall availability is the product: 0.999^N — availability degrades with every synchronous dependency you add. Redundant components in parallel raise availability: if a component has availability A and you run 2 independent replicas, effective availability of "at least one is up" is 1 - (1-A)^2. This is the mathematical justification for redundancy and for minimizing serial dependencies.

### SLA Targets — The "Nines" Table

Availability is conventionally expressed in "nines" — the percentage of time a system is up over a year, and the corresponding allowed downtime:

| Availability | Downtime/year | Downtime/month | Downtime/week | Common tier |
|---|---|---|---|---|
| 99% ("two nines") | 3.65 days | 7.2 hours | 1.68 hours | Internal tools |
| 99.9% ("three nines") | 8.76 hours | 43.2 minutes | 10.1 minutes | Standard SaaS SLA |
| 99.95% | 4.38 hours | 21.6 minutes | 5.04 minutes | Mid-tier cloud SLA |
| 99.99% ("four nines") | 52.6 minutes | 4.32 minutes | 1.01 minutes | Payment/e-commerce critical path |
| 99.999% ("five nines") | 5.26 minutes | 25.9 seconds | 6.05 seconds | Telecom, core banking |

Practical implications: going from three to four nines is not a linear engineering cost — it roughly requires eliminating single points of failure, automating all failover (no human in the loop), and often multi-region active-active. Each additional nine multiplies engineering and operational cost by a large factor, so SLA targets should be set per-component based on business impact, not blanket 99.99% everywhere. A system's *effective* availability is bounded by its least-available critical-path dependency, so SLA composition (per the multiplication rule above) must be checked end-to-end, not per-service.

### Bulkhead Pattern

Named after ship hull compartments that stop one breach from sinking the whole vessel. In software, the bulkhead pattern **isolates resource pools** (thread pools, connection pools, CPU/memory quotas, or entire service instances) per dependency or per tenant, so that a failure or slowness in one does not exhaust shared resources and cascade into unrelated failures.

- **Thread-pool bulkhead**: each downstream call (e.g., payments, inventory, recommendations) gets its own bounded thread pool/semaphore rather than sharing one pool. If the recommendations service starts responding slowly, its pool fills and blocks — but the payments pool is untouched and continues serving.
- **Process/container bulkhead**: run noisy or high-risk tenants/workloads in separate deployments so a memory leak or crash loop in one doesn't take down others (physical isolation vs logical isolation, at higher infra cost).
- **Data-partition bulkhead**: shard by tenant/entity (e.g., per hot auction) so that load or lock contention on one partition can't starve others.
- Bulkhead is often paired with a **circuit breaker**: the bulkhead limits concurrent damage, the circuit breaker stops sending requests to a dependency that's clearly failing, giving it time to recover and avoiding wasted retries.

### Timeout Strategies — Fail Fast vs Wait-and-Retry

Every network call needs a timeout budget; the choice between failing fast and retrying is a core resilience trade-off.

- **Fail fast**: set an aggressive timeout (tuned to p99 latency of the healthy dependency, not the average) and return an error or fallback immediately. Frees up threads/connections quickly, prevents backpressure from propagating upstream, and gives a fast, predictable user experience. Downside: transient blips that would have resolved in 50ms now surface as user-visible errors.
- **Wait-and-retry**: retry the call, ideally with **exponential backoff and jitter** (to avoid synchronized retry storms across clients — the classic "thundering herd") and a capped retry budget (e.g., 2-3 attempts, overall deadline). Good for idempotent operations against dependencies with mostly-transient failures. Downside: retries multiply load on an already struggling dependency (retry amplification) and increase tail latency for the caller.
- **Combining them correctly**: use a short per-attempt timeout, a small number of retries with backoff+jitter, an overall request deadline propagated down the call chain (so a client that gave up doesn't leave orphaned work running server-side), and a circuit breaker to stop retrying altogether once failure rate crosses a threshold. Never retry non-idempotent operations (e.g., "place bid") without an idempotency key, or you risk duplicate side effects.

### Graceful Degradation and Fallback Responses

Graceful degradation means a service under partial failure serves a **reduced but still useful** experience rather than a hard error. Design principles:

- Distinguish **critical path** (must succeed — e.g., recording that a bid was accepted) from **best-effort path** (can degrade — e.g., live notification of a new high bid, personalized recommendations, view counts).
- Provide **fallback responses**: cached/stale data, a default value, or a simplified view, instead of propagating the failure. Example: if a recommendation service is down, show generic best-sellers instead of an error page.
- **Feature flags / load shedding**: under extreme load, proactively disable non-essential features (e.g., "recently viewed" widgets) to protect core functionality — this is a deliberate, pre-planned degradation, not an accident.
- Make degraded state observable (metrics/alerts) so it's a visible, temporary trade-off, not a silent data-quality regression.

### Disaster Recovery Planning

Disaster Recovery (DR) planning covers surviving large-scale failures — a full region outage, data corruption, ransomware — that redundancy within one region cannot address. Key elements:

- **DR strategies**, in increasing cost and decreasing RTO/RPO: **backup & restore** (cheapest, RTO/RPO in hours), **pilot light** (minimal always-on core infra in the DR region, scale up on failover), **warm standby** (scaled-down but fully functional replica, promoted on failover), **hot/multi-site active-active** (full capacity running in multiple regions simultaneously, near-zero RTO/RPO, highest cost).
- A DR plan documents: failure scenarios in scope, RTO/RPO targets per system tier, runbooks for failover/failback, data replication topology, and — critically — a **regular test cadence** (DR drills / game days), since an untested DR plan is not a plan.
- Failback (returning to the primary region after it recovers) is as important to plan as failover, and often neglected.

### Chaos Engineering

Chaos engineering is the discipline of proactively injecting failure into a system (in production or production-like environments) to verify it actually behaves the way its resilience design assumes. Netflix pioneered this at scale:

- **Chaos Monkey** (2011) randomly terminates production instances during business hours, forcing engineers to build services that tolerate instance loss as a matter of course rather than an occasional surprise.
- **Chaos Kong** simulates the loss of an entire AWS region, testing Netflix's multi-region failover capability — a scale of test far beyond a single-instance kill.
- The broader **Simian Army** extended this to Latency Monkey (injects artificial delays), Conformity Monkey (flags instances not following best practices), and others, later generalized into the open-source **Chaos Automation Platform (ChAP)**.
- Principles: run experiments in production (staging rarely has representative traffic/scale), start with a steady-state hypothesis ("this metric should stay within X"), minimize blast radius (small % of traffic first), and automate rollback the instant the hypothesis is violated. This turns resilience claims into continuously-verified facts instead of assumptions.

### RTO and RPO

Two metrics that quantify disaster-recovery requirements and directly drive architecture choices:

- **RTO (Recovery Time Objective)**: the maximum acceptable time to restore service after a disruption. Answers "how long can we be down?" A low RTO (minutes) forces investment in automated failover and warm/hot standby infrastructure; a high RTO (hours) permits cheaper cold backup-and-restore approaches.
- **RPO (Recovery Point Objective)**: the maximum acceptable amount of data loss, measured as time. Answers "how much data can we afford to lose?" A low RPO (seconds) requires synchronous or near-synchronous cross-region replication; a high RPO (hours) permits nightly backups.
- They are independent dials: a system can have a low RTO but high RPO (fails over fast, to slightly stale data) or vice versa (takes a while to come back, but loses nothing). Business/financial-transaction systems (like bidding) typically demand low RPO (can't lose a committed bid) even if RTO is a few minutes, because data loss is often unrecoverable while downtime is merely costly.
- RTO/RPO targets should be set per data class/service tier (e.g., "auction bid ledger: RPO 0, RTO 2 min" vs "recommendation cache: RPO 1 hour, RTO 30 min"), not uniformly, because the cost of tightening either dial rises sharply near zero.

---

## Case Study Solution: Online Auction Platform

### Problem Statement & Clarifying Requirements

Design the backend for a live online auction platform (eBay-style): sellers list items with a start time, end time, and starting price; buyers place bids in real time; the platform must never lose an accepted bid and must never let two buyers simultaneously "win" the same item (no double-selling), especially under the extreme bid-rate spike in an auction's final seconds.

**Functional requirements**
- Create/list an auction (item, start price, start/end time, optional reserve/buy-now).
- Place a bid on an active auction; a bid is accepted only if it exceeds current highest bid (+ minimum increment) and the auction is still open.
- Subscribe to real-time auction updates (current price, time remaining, outbid notifications).
- Close an auction at its end time and determine the winner unambiguously.

**Non-functional requirements**
- **Strict bid ordering / no lost bids**: bids for a given auction must be totally ordered and durably recorded; the system must be linearizable per-auction even though it's distributed.
- **High availability during peak bidding**, particularly the last seconds of a hot auction (bid-rate spike of 100-1000x baseline).
- **Correctness under partial failure**: a crashed node, a network partition, or a downstream (notification) outage must never cause a lost bid, a double winner, or an inconsistent final price.
- Low latency bid acknowledgment (target p99 < 300ms) so bidders get fast confirm/reject feedback near the deadline.
- Auditability: full immutable bid history per auction for dispute resolution.

### Capacity Estimation

Assume a platform with 5M DAU, 200K live auctions at any moment, and a long-tail popularity distribution where a small number of "hot" auctions absorb most late bidding.

- **Baseline bid volume**: ~2M bids/day average → ~23 bids/sec average, trivial load.
- **Peak — last minute of a hot auction**: real auction data (and eBay's own documented "sniping" behavior) shows a large fraction of bids on a popular item arrive in the closing seconds. Model a hot auction receiving 500 bids in its final 60 seconds → ~8-10 bids/sec sustained for that single auction, bursting to 50+ bids/sec in the final 2-3 seconds as competing sniping bots race the clock.
- **Platform-wide peak**: with 500 auctions closing concurrently in a busy hour (e.g., end-of-day mass closes) each contributing a similar tail spike, aggregate peak bid ingestion ≈ 500 × 10 bids/sec ≈ 5,000 bids/sec platform-wide, roughly 200x the average rate — this is the number the ingestion tier must be provisioned for, not the daily average.
- **Read/notification load**: each active bidder on a hot auction polls or holds a live connection for price updates; a hot auction with 2,000 concurrent watchers broadcasting on every bid (10/sec) generates ~20,000 update pushes/sec for that one auction alone — this is why notification fan-out must be decoupled and independently scalable (and allowed to degrade) from bid ingestion.
- **Storage**: a bid record (~200 bytes) × 2M bids/day ≈ 400MB/day of append-only bid data — cheap to store durably indefinitely for audit purposes.

### High-Level Architecture

```
                        ┌─────────────────────┐
                        │   Client (Web/App)   │
                        └─────────┬────────────┘
                                  │ HTTPS / WebSocket
                        ┌─────────▼────────────┐
                        │   API Gateway / LB    │  (multi-AZ, health-checked)
                        └───┬───────────────┬───┘
                            │               │
                 ┌──────────▼───┐    ┌──────▼────────────┐
                 │ Bid Ingestion │    │ Subscription /     │
                 │   Service     │    │ Notification Svc   │
                 │ (stateless,   │    │ (WebSocket/SSE,     │
                 │  multi-AZ,    │    │  bulkheaded per     │
                 │  N replicas)  │    │  auction shard)     │
                 └──────┬────────┘    └────────┬────────────┘
                        │  per-auction routed         ▲
                        │  (consistent hash on         │ pub/sub fan-out
                        │   auction_id)                │ (Kafka/Redis Streams)
                        ▼                              │
              ┌───────────────────────┐                │
              │ Auction State Service │────────────────┘
              │ (single logical writer│
              │  per auction_id via   │
              │  partition ownership; │
              │  in-memory + WAL)     │
              └──────────┬────────────┘
                          │ synchronous append + ack
                          ▼
              ┌───────────────────────────┐
              │ Bid Ledger (durable store) │
              │ Primary (AZ-1) ── sync ──▶ Replica (AZ-2)
              │                    async ─▶ Replica (Region-2, DR)
              └───────────────────────────┘
```

- **Bid Ingestion Service**: stateless, horizontally scaled behind the LB across multiple AZs. Validates request shape and auth, then routes each bid deterministically (consistent hashing on `auction_id`) to the **Auction State Service** instance owning that auction's partition.
- **Auction State Service**: the linearizability boundary. Exactly one active writer (leader) owns each auction partition at a time, giving a natural **single-writer-per-auction** design — this avoids distributed-transaction complexity for the hot path. Ownership is managed via a consensus/lease mechanism (e.g., a Raft-based coordinator or a lease row in a strongly consistent store) so failover promotes exactly one new leader, never two.
- **Bid Ledger**: an append-only durable log (e.g., a partitioned relational store or log-structured store) with synchronous replication to a second AZ (for RPO≈0 within region) and asynchronous replication to a DR region.
- **Notification Service**: subscribed to a pub/sub stream of state changes (Kafka/Redis Streams) and fans out to WebSocket/SSE clients. Decoupled from the write path — its failure never blocks bid acceptance (see Deep Dive).
- **Multi-AZ / redundancy**: every stateless tier runs ≥3 replicas across ≥2 AZs; the Bid Ledger's primary/standby span AZs with automatic failover; a second region holds a warm-standby replica for DR.

### API Design

```
POST /v1/auctions/{auction_id}/bids
Headers: Idempotency-Key: <client-generated UUID>
Body: { "bidder_id": "u123", "amount": 152.50 }

200 OK
{ "bid_id": "b-9f2a", "status": "accepted", "current_high": 152.50,
  "auction_version": 4187 }

409 Conflict
{ "status": "rejected", "reason": "outbid",
  "current_high": 155.00, "min_next_bid": 157.50 }

410 Gone
{ "status": "rejected", "reason": "auction_closed", "winning_bid": 160.00 }
```
- `Idempotency-Key` makes retried bid submissions safe (client or gateway may retry on timeout without risking a duplicate bid).
- `auction_version` is an optimistic-concurrency token; a client can pass `If-Match: auction_version` to implement compare-and-swap semantics client-side if desired, though the server is authoritative regardless.

```
GET /v1/auctions/{auction_id}/stream    (Server-Sent Events or WebSocket upgrade)

event: price_update
data: { "auction_id": "a-77", "current_high": 152.50, "ends_at": "...", "auction_version": 4187 }

event: auction_closed
data: { "auction_id": "a-77", "winning_bid": 160.00, "winner_id": "u456" }
```
- Subscription is best-effort and explicitly allowed to lag or drop under load (see graceful degradation below) — it is never the source of truth for whether a bid was accepted; the synchronous `POST /bids` response is.

### Data Model

```sql
-- One row per auction; version is the optimistic-concurrency guard.
CREATE TABLE auctions (
  auction_id      UUID PRIMARY KEY,
  seller_id       UUID NOT NULL,
  item_id         UUID NOT NULL,
  start_price     NUMERIC(12,2) NOT NULL,
  reserve_price    NUMERIC(12,2),
  min_increment   NUMERIC(12,2) NOT NULL DEFAULT 1.00,
  current_high_bid NUMERIC(12,2),
  current_high_bidder UUID,
  starts_at       TIMESTAMPTZ NOT NULL,
  ends_at         TIMESTAMPTZ NOT NULL,
  status          TEXT NOT NULL CHECK (status IN ('scheduled','open','closed','cancelled')),
  version         BIGINT NOT NULL DEFAULT 0   -- optimistic lock / auction_version
);

-- Append-only, immutable ledger — the source of truth for bid history and disputes.
CREATE TABLE bids (
  bid_id          UUID PRIMARY KEY,
  auction_id      UUID NOT NULL REFERENCES auctions(auction_id),
  bidder_id       UUID NOT NULL,
  amount          NUMERIC(12,2) NOT NULL,
  submitted_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  server_sequence BIGINT NOT NULL,             -- monotonic, assigned by the single owner
  idempotency_key TEXT NOT NULL,
  outcome         TEXT NOT NULL CHECK (outcome IN ('accepted','rejected_outbid','rejected_closed')),
  UNIQUE (auction_id, idempotency_key)
);
CREATE UNIQUE INDEX idx_bids_auction_seq ON bids(auction_id, server_sequence);
```
- **Concurrency control**: two complementary mechanisms. (1) The Auction State Service enforces a **single-writer-per-auction** invariant — only the current partition leader mutates `auctions.current_high_bid`/`version`, so within that scope no compare-and-swap race is even possible. (2) As defense in depth (e.g., a direct DB write path or leader-handoff edge case), updates use **optimistic locking**: `UPDATE auctions SET current_high_bid=?, version=version+1 WHERE auction_id=? AND version=? AND ? > current_high_bid`, retried on the app side if `version` mismatched. `server_sequence` gives every bid a strict total order per auction, independent of client clock skew, which is what actually prevents "lost bids" — the ledger, not wall-clock time, defines who bid first.
- The `UNIQUE (auction_id, idempotency_key)` constraint makes bid submission safe to retry.

### Deep Dive

**Bulkhead pattern applied**: auctions are partitioned (consistent hashing on `auction_id`) across N Auction State Service shards, each with its own thread pool, connection pool, and in-memory working set. A single pathologically hot auction (viral item, bot storm) can saturate its own shard's resources without starving the shards serving the other 199,999 concurrent auctions. The Notification Service is bulkheaded the same way — per-auction (or per-shard) fan-out queues and connection pools, so a hot auction with 20,000 watchers backing up its notification queue cannot delay notification delivery for unrelated auctions. Additionally, the read path (auction browsing/search) and write path (bid placement) run on entirely separate service tiers and connection pools, so a browsing traffic spike cannot degrade bid-acceptance latency.

**Timeout strategy for bid placement**: this is a non-idempotent-feeling but idempotency-keyed financial action, so the strategy favors **fail fast with a bounded, non-blind retry**, not aggressive retry-and-wait. Client → Gateway → Ingestion timeout budget: 250ms per hop, ~800ms end-to-end deadline. Ingestion → Auction State Service: single attempt at 150ms timeout; on timeout, **do not blindly retry** the write (it may have already committed) — instead the client retries with the *same* `Idempotency-Key`, and the ledger's unique constraint guarantees at-most-once application even if the original request actually succeeded server-side. This converts an unsafe "wait and retry" into a safe fail-fast-and-client-resubmits pattern. A circuit breaker on the Ingestion→State path trips after a sustained error rate on a shard, immediately returning `503 Service Unavailable` (clearly distinguishable from a business rejection) rather than queueing requests that will only time out anyway — critical in the last seconds of an auction where a queued-then-timed-out bid is effectively a lost bid.

**Graceful degradation if the Notification Service is down**: bid acceptance never depends on notification delivery — the `POST /bids` call only touches Ingestion and the Auction State Service/Ledger; publishing to the notification stream is fire-and-forget, off the critical path, done asynchronously after the ledger write is durably acknowledged. If the Notification Service (or its pub/sub backbone) is fully down: bids continue to be accepted and recorded normally; clients simply stop receiving live price-update pushes and fall back to polling `GET /auctions/{id}` (a cheap, cacheable read against the Auction State Service or a read replica) at a modest interval; the UI shows a "live updates paused" indicator. This is a deliberate, pre-planned degraded mode, not a failure — correctness (no lost bids) is fully preserved even with zero notification capacity.

**RTO/RPO targets**:
- **Bid Ledger**: **RPO = 0** (synchronous replication within region before ack) — an accepted bid is a financial commitment; losing one is unacceptable and directly causes disputes/double-selling risk. **RTO = 2 minutes** within-region (automated leader failover via the partition-ownership lease) — a short user-visible interruption is tolerable, data loss is not.
- **Cross-region DR** (regional outage): **RPO ≈ 5-15 seconds** (async cross-region replication lag) — a full regional failure may lose the last few seconds of bids platform-wide, which is disclosed and reconciled via the immutable ledger's audit trail (e.g., extending affected auctions' close time), an accepted trade-off given the cost of synchronous cross-region writes on every bid. **RTO = 15 minutes** for full regional failover (warm standby, scale-up + DNS/traffic cutover).
- **Notification Service**: **RPO/RTO essentially unconstrained** (best-effort, stateless, can be rebuilt/restarted with no data-loss implications) — justified precisely because it's off the correctness-critical path.

### Trade-offs and Alternatives Considered

- **Single-writer-per-auction vs distributed consensus per bid**: an alternative is running every bid through a distributed transaction/consensus protocol (e.g., Paxos/Raft-backed compare-and-swap on every write) without partition ownership. Rejected as the primary mechanism because it adds latency and complexity to every single bid; single-writer-per-partition gets the same linearizability with much lower per-request cost, at the cost of a short unavailability window during leader failover (mitigated by keeping that window sub-2-seconds via fast lease expiry).
- **Optimistic locking alone (no partition ownership)**: simpler to build, but under true peak load (50+ bids/sec on one auction) generates heavy contention and wasted retries right when correctness matters most (auction close). Kept as a defense-in-depth layer, not the primary mechanism.
- **Extending auction close time on late bids ("soft close")**: many real auction platforms (and explicitly eBay's design philosophy, which instead tolerates sniping) face this choice. eBay deliberately does *not* auto-extend, accepting "sniping" as an intended dynamic of a hard-deadline, sealed-final-moment auction — this is a product decision, not a hard system-design requirement, but it simplifies the backend (no dynamic deadline mutation under load) and shaped this design's assumption of a fixed `ends_at`.
- **Notification via polling vs push**: push (WebSocket/SSE) gives lower latency and less wasted request volume than polling but adds substantial connection-management infrastructure; the design uses push as primary with polling as the degraded fallback, capturing most of the benefit while keeping a simple fallback path.

### How Real Systems Solve This

Auction and e-commerce platforms handling bidding at scale converge on similar ideas documented publicly: strict server-authoritative ordering of bids independent of client-reported time (to prevent clock-skew cheating and disputed "who bid first" claims), fraud/anomaly detection layered onto the bid pipeline (eBay's real-time systems apply bidding-pattern and fraud checks inline with acceptance, per public write-ups on their auction and fraud-prevention stack), and hard acceptance of last-second bidding ("sniping") as a designed-for load pattern rather than an edge case to eliminate. On the resilience-engineering side, Netflix's Chaos Monkey/Chaos Kong practice — randomly killing instances and even whole AWS regions in production to verify failover actually works — is the industry-reference approach for validating that a bulkhead/failover design like the one above behaves correctly under real, not simulated, failure; the same game-day discipline (e.g., killing an Auction State Service shard leader mid-peak-bidding-window in a controlled experiment) is the right way to validate this design's failover claims before trusting them in production.

## Sources

- [Chaos Engineering Upgraded (Chaos Kong) — Netflix TechBlog](http://techblog.netflix.com/2015/09/chaos-engineering-upgraded.html)
- [Chaos Monkey at Netflix: the Origin of Chaos Engineering — Gremlin](https://www.gremlin.com/chaos-monkey/the-origin-of-chaos-monkey)
- [Resilience in Microservices: Bulkhead vs Circuit Breaker — Medium](https://medium.com/@parserdigital/resilience-in-microservices-bulkhead-vs-circuit-breaker-54364c1f9d53)
- [Microservices Resilience Patterns — GeeksforGeeks](https://www.geeksforgeeks.org/system-design/microservices-resilience-patterns/)
- [Circuit Breaker, Bulkhead, and Retry Patterns Explained — scalewithchintan.com](https://scalewithchintan.com/blog/circuit-breaker-bulkhead-retry-patterns-demystified)
- [Inside eBay's Real-Time Auction System: Bidding Logic, Algorithms & Fraud Prevention Techniques — Frugal Testing](https://www.frugaltesting.com/blog/inside-ebays-real-time-auction-system-bidding-logic-algorithms-fraud-prevention-techniques)
- [Bid sniping — eBay Help](https://www.ebay.com/help/buying/bidding/bid-sniping?id=4224)
- [Design Online Auction: A Complete Guide — System Design Handbook](https://www.systemdesignhandbook.com/guides/design-online-auction/)
- [RTO vs RPO: Key Differences in Disaster Recovery Planning — SentinelOne](https://www.sentinelone.com/cybersecurity-101/cloud-security/rto-vs-rpo/)
- [Recovery Point Objective (RPO) vs. Recovery Time Objective (RTO) — Splunk](https://www.splunk.com/en_us/blog/learn/rpo-vs-rto.html)
- [What Is the Difference Between RTO and RPO? — Rubrik](https://www.rubrik.com/insights/rto-rpo-whats-the-difference)
