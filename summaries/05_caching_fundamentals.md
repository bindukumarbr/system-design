# Module 05: Caching Fundamentals

## Core Concepts

### Caching Patterns
1. **Cache-Aside (Lazy Loading):** App checks cache -> Miss -> Queries DB -> Updates Cache. Safest and most common pattern.
2. **Read-Through:** App asks Cache Library -> Library fetches from DB on miss. Centralizes logic.
3. **Write-Through:** App writes to Cache -> Cache writes to DB synchronously. Great read consistency, high write latency.
4. **Write-Behind (Write-Back):** App writes to Cache -> Cache async flushes to DB. High throughput, but risks data loss if cache crashes. **NEVER use for financial/inventory data.**

### Eviction Policies
- **LRU (Least Recently Used):** Good default. Evicts oldest accessed.
- **LFU (Least Frequently Used):** Good for stable hot sets, avoids evicting popular items during random scans.
- **FIFO (First In First Out):** Ignores usage frequency, simple.
- *Note:* TTL controls staleness; Eviction controls memory capacity.

### Cache Stampede (Thundering Herd) Prevention
When a hot key expires and 1000 requests hit the DB at once:
1. **Mutex/Locks:** Only 1 request queries DB; others wait or get stale data.
2. **Request Coalescing (Single-Flight):** App collapses concurrent requests into one backend call.
3. **Probabilistic Early Expiration (XFetch):** Randomly refresh a key *before* it expires to spread load.
4. **Jitter:** Add randomness to TTLs so they don't all expire at exactly the same millisecond.

### When NOT to Cache
- Strictly consistent, highly contended counters (e.g., Ticketmaster seat inventory, account balances). **Do not cache mutable authority.**

---

## Case Study: Ticketmaster
- **Requirements:** Flash sales (1000x traffic spikes in minutes), NO overselling, hold seats for 10 minutes.
- **Estimations:** 2M users hitting the site -> 10,000 requests/sec. Seat inventory is highly contended.
- **Architecture:**
  - **Virtual Waiting Room:** Acts as a queue/throttle. Converts 10,000 req/s to a manageable 50 req/s.
  - **Cache-Aside (Redis):** Serves static metadata (event details, venue layout). Highly cacheable.
  - **Seat Inventory Store (ACID DB):** Source of truth. NEVER CACHE SEAT STATUS AUTHORITATIVELY.
- **Aha! Insights:**
  - **Optimistic Concurrency Control:** Instead of pessimistic row locks (which block the DB), use version checks (`UPDATE ... WHERE version=X`). If someone else bought it, version changes, and the update fails safely.
  - The waiting room is the most important component; you cannot out-scale a flash sale, you must shape the traffic.
