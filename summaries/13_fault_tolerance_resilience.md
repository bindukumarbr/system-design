# Module 13: Fault Tolerance & Resilience

## Core Concepts

### High Availability (HA) & Redundancy
- **Availability = Redundancy + Failover.** 
- If you have components in series (A calls B calls C), overall availability drops. 
- If you have components in parallel (Load Balancer -> App1, App2), overall availability increases.
- **Active-Active:** All replicas take traffic. Fast failover.
- **Active-Passive:** One takes traffic, the other waits. Takes longer to failover.

### SLA Targets (The "Nines")
- **99.9% (Three Nines):** ~43 mins downtime/month. Standard SaaS.
- **99.99% (Four Nines):** ~4.3 mins downtime/month. Payment critical path.
- **99.999% (Five Nines):** ~26 secs downtime/month. Core banking.
- Going from 3 nines to 4 nines is expensive; it requires automating all failovers and often multi-region architectures.

### Bulkhead Pattern
- Named after ship hulls. Prevent one failure from sinking the whole ship.
- **Example:** Give the `Payments` service its own thread pool, and `Recommendations` its own. If `Recommendations` hangs and consumes all its threads, `Payments` is unaffected.

### Timeout Strategies & Retries
- **Fail Fast:** Set short timeouts based on p99 latency. Protects threads but exposes transient errors to users.
- **Wait and Retry:** Must use **Exponential Backoff and Jitter**. Jitter prevents a thundering herd where 1000 failed clients all retry at the exact same millisecond.
- Combine them: short timeouts, bounded retries with jitter, and an overall request deadline.

### Chaos Engineering
- Originated by Netflix (Chaos Monkey, Chaos Kong). Proactively killing instances in production to prove that failover actually works, rather than just assuming it works.

### RTO vs RPO
- **RTO (Recovery Time Objective):** How long can we be down? (Drives hot-standby vs cold-backup decisions).
- **RPO (Recovery Point Objective):** How much data can we lose? (Drives synchronous vs asynchronous replication decisions).

---

## Case Study: Online Auction Platform
- **Requirements:** Live bidding (eBay style). High burst traffic in the last 60 seconds of a popular auction. Cannot lose a bid. Cannot double-sell an item.
- **Estimations:** Peak traffic (5,000 bids/sec across hot auctions) is vastly higher than average traffic (23 bids/sec).
- **Architecture:** 
  - **Bid Ingestion:** Horizontally scaled API tier.
  - **Auction State Service:** Uses a **Single-Writer-Per-Partition** pattern. Each auction ID hashes to exactly one active leader instance. This prevents race conditions natively.
  - **Bid Ledger:** A durable append-only log, replicated synchronously within the region (RPO = 0) and asynchronously out of region (RPO ~ 5s).
- **Aha! Insights:**
  - **Graceful Degradation:** The Notification (WebSocket) service is decoupled. If it crashes, users just don't see live price updates and fall back to polling. The actual bid placement is untouched.
  - **Optimistic Locking:** If two people bid at the exact same millisecond, the database uses a `version` integer to ensure only one wins the update (`UPDATE auctions SET price=?, version=version+1 WHERE version=?`).
  - **Idempotency Keys:** If a client doesn't get a response, they retry with the exact same `Idempotency-Key` UUID. The ledger ensures this is not double-charged.
