# Module 06: Distributed Caching & CDN

## Core Concepts

### Redis vs Memcached
- **Memcached:** Simple blob cache. Multi-threaded per node. Use for massive, simple HTML fragment or DB row caching.
- **Redis:** Rich data structures (Lists, ZSETs, Hashes). Single-threaded per core. Use when caching has structure (e.g., leaderboards, rate limiters, timelines).

### Redis Cluster
- Splits keyspace into 16,384 **hash slots**.
- **Important:** Redis Cluster is **NOT strongly consistent**. It uses async primary->replica replication. Do not use as a system of record if data loss is unacceptable.
- **Hash Tags:** Use `{userId}:feed` to force related keys to hash to the same node for atomic multi-key operations.

### Multi-Tier Caching
- **L1 (In-Process):** Fast (nanoseconds), local to the app instance. Hard to invalidate across fleet.
- **L2 (Distributed - Redis):** Shared across fleet. Milliseconds. Where most derived data lives.
- **L3 (CDN / Edge):** Physically close to users. Caches static assets (images, JS).

### CDN Cache-Control Headers
- `max-age`: Time to live.
- `no-cache`: Forces cache to revalidate with origin before serving. (Does *not* mean "don't cache").
- `no-store`: Do not cache anywhere.
- `stale-while-revalidate`: Serve stale content to user while fetching fresh content async in the background (huge UX win).

---

## Case Study: Facebook News Feed
- **Requirements:** 1B DAU, ranked feed of friends/pages, fast reads, massive fan-out skew (celebrities vs normal users).
- **Estimations:** 5B feed reads/day (58k/sec) vs 500M posts/day (6k/sec).
- **Architecture:**
  - **Fan-Out-On-Write (Push):** When a normal user posts, push the Post ID into all followers' precomputed Redis ZSETs (Sorted Sets).
  - **Fan-Out-On-Read (Pull):** When a celebrity posts, DO NOT push to 50M followers (write amplification disaster). Instead, pull their posts at read-time and merge.
  - **Hybrid Approach:** The industry standard. Push for small networks, Pull for celebrities.
  - **Redis ZSETs:** Perfect for feeds. `score` is rank/timestamp, `member` is Post ID. Use `ZREVRANGE` for pagination.
- **Aha! Insights:**
  - **Opaque Cursors for Pagination:** Do not use `OFFSET/LIMIT` for feeds, because if a new post is added, the offset shifts and items are duplicated. Use an opaque cursor (like the last seen timestamp or score).
  - Use Memcached for the actual Post objects (dumb blobs), and Redis for the Feed Index (ZSETs).
