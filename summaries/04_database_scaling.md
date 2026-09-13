# Module 04: Database Scaling

## Core Concepts

### Sharding (Horizontal Partitioning)
- **Range-based:** Split by contiguous keys (e.g., User 1-1000). Good for range scans, but prone to hotspotting (e.g., if key is timestamp, new shard gets all writes).
- **Hash-based:** Split by `hash(key) % N`. Eliminates hotspots, but destroys range query capability. Adding a node requires reshuffling everything (unless using Consistent Hashing).
- **Directory-based:** A lookup service holds the exact mapping. Flexible, but adds a network hop and a single point of failure.

### Consistent Hashing
- **The Problem:** Modulo hashing (`hash % N`) breaks when N (shard count) changes.
- **The Solution:** Hash both nodes and keys onto a fixed circular ring. A key belongs to the first node encountered walking clockwise. Adding/removing a node only affects its immediate neighbor, moving `1/N` of the data.
- **Virtual Nodes:** Assign each physical shard to multiple points on the ring to balance the load evenly.

### Read/Write Separation (Primary-Replica)
- **Pattern:** All writes go to the Primary. It streams a replication log to Read Replicas.
- **Problem (Replication Lag):** Async replication means a replica might be milliseconds/seconds behind.
- **Solutions to Lag:** Route a user's reads to the Primary for a short window after they write ("Read Your Own Writes"), or use sticky routing.

### Multi-Leader Replication & Conflict Resolution
- Allows writes at multiple nodes (e.g., across regions) for lower write latency, but introduces conflicts.
- **Last-Write-Wins (LWW):** Highest timestamp wins. Simple, but drops data. (Cassandra/Dynamo).
- **Vector Clocks:** Tracks causality; surfaces conflicts to the app to merge manually. (Dynamo cart).
- **CRDTs:** Data structures (like PN-Counters) that merge deterministically and mathematically.

---

## Case Study: News Aggregator (Reddit / Hacker News)
- **Requirements:** 50M DAU, upvoting, ranking, 200k new articles/day. Extremely read-heavy.
- **Estimations:** 30,000 reads/sec vs ~100 writes/sec (1000:1 read/write ratio).
- **Architecture:**
  - **Ingestion Pipeline:** Async queue for decoupling flaky external RSS feeds.
  - **Sharded DB:** Hash-partitioned by `article_id`. Co-locates votes/comments with the article.
  - **Ranking Service:** Background job that periodically recomputes scores, avoiding live cross-shard aggregations.
- **Aha! Insights:**
  - **Why not shard by Source?** Because sources are highly skewed (hotspotting). Hashing by `article_id` guarantees even write/read distribution.
  - **Precomputed Hot Scores:** Do not compute Reddit's `log10(votes) + age` on the fly. A background cron job computes this and caches it.
  - **Connection Pooling is mandatory:** To fan out 30k reads/sec across hundreds of app instances to the read replicas, you must use a pooler (PgBouncer) to avoid exhausting DB connections.
