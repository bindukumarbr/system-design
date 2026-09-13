# Module 01: System Design Thinking

## Core Concepts & Frameworks

### HLD vs. LLD

- **High-Level Design (HLD):** Architecture-level view (services, DBs, caches, flow). No code. Default to this in interviews.
- **Low-Level Design (LLD):** Implementation-level (class diagrams, schemas, specific algorithms). Only dive into this for the "Distinctive Component" (e.g., ID generation algorithm).

### Functional vs. Non-Functional Requirements (FRs vs NFRs)

- **Functional (FRs):** What the system does (APIs, core use cases).
- **Non-Functional (NFRs):** How well it does it (Scalability, Availability, Latency, Consistency). **Crucial:** NFRs drive architecture decisions. Do not skip them.

### PEDALS Framework (Primary Interview Structure)

- **P - Process Requirements:** Clarify FRs, NFRs, and scope constraints.
- **E - Estimate:** Back-of-envelope math (Capacity, QPS, Storage, Bandwidth).
- **D - Design:** High-level architecture components and flow.
- **A - APIs:** Define the contract (REST, gRPC, WebSocket).
- **L - Latency & Availability:** Load balancing, caching, CDNs.
- **S - Scale:** Sharding, partitioning, async queues.

_(Note: RESHADED is a similar alternative that explicitly adds an **Evaluation** step to loop back to requirements at the end, and a **Distinctive Component** step to highlight advanced knowledge)._

### Capacity Estimation (Back-of-Envelope Math)

1. **DAU → Daily Events:** DAU × actions-per-user.
2. **Average QPS:** Daily Events / 86,400.
3. **Peak QPS:** Avg QPS × Peak Factor (usually 2x–5x).
4. **Storage:** Writes/day × bytes/record × retention. Add 2–3x for indexes/replication.
5. **Bandwidth:** QPS × avg payload size.

### Bottleneck Analysis

Identify the constraint _before_ designing:

- **Compute-bound:** Horizontal scaling, stateless services.
- **Read-bound / Hot-key:** Caching, read replicas.
- **Write-bound:** Sharding, async processing, write-ahead batching.
- **Storage-bound:** Tiering (hot/warm/cold), compression.
- **Coordination-bound:** Batching, hashing to remove central locks.

### CAP Theorem & PACELC

- **CAP:** Pick Consistency or Availability during a network Partition.
- **PACELC:** If Partition, pick A or C; Else (normal operation), pick Latency or Consistency.
- _Tip:_ Apply these per-subsystem, not to the whole architecture. (e.g., user profiles = AP, financial transactions = CP).

---

## Case Study: URL Shortener

- **Requirements:** 100M DAU, extremely read-heavy (100:1 read/write ratio), low latency (<100ms). Uniqueness guaranteed.
- **Estimations:** 350 QPS Writes vs 35,000 QPS Reads. Bandwidth is low (21 MB/s). **Bottleneck:** Read throughput/latency.
- **Architecture:**
  - API Gateway → Write Service (reserves batched IDs) → Cache → DB.
  - API Gateway → Redirect Service → Cache (hit) or DB (miss).
- **Distinctive Component (ID Generation):** Counter-based (e.g., batched Redis INCR) chosen over Hash-based to completely avoid collision retry loops, despite producing guessable IDs.
- **Aha! Insights:**
  - Use **302 (Temporary) redirects** over 301 (Permanent). 301s cache in the browser forever, breaking analytics/click tracking.
  - The system is read-heavy, so a **Cache-first architecture** with Redis/Memcached is justified by the math.
  - Relational vs NoSQL doesn't matter much for simple KV lookups at this scale, but NoSQL naturally scales horizontally for this access pattern.
