# Module 03: Database Fundamentals

## Core Concepts

### RDBMS vs NoSQL
- **RDBMS (SQL):** Use for strong consistency, stable schema, multi-row ACID transactions (payments, inventory).
- **NoSQL:** Use for massive horizontal write scale, eventual consistency, evolving schemas (user profiles, catalogs).
- *Insight:* Most real architectures are **polyglot** (e.g., SQL for payments + Redis for caching + ElasticSearch for text search).

### ACID vs BASE
- **ACID (SQL):** Atomicity, Consistency, Isolation (via MVCC/locks), Durability.
- **BASE (NoSQL):** Basically Available, Soft state, Eventual consistency. Good when business logic tolerates staleness (e.g., view counts, presence).

### PACELC Theorem (Extends CAP)
- **CAP:** If Partition, pick Availability (AP) or Consistency (CP).
- **PACELC:** Even without a partition, you must trade Latency (L) vs Consistency (C).
- *Example:* DynamoDB defaults to PA/EL (favors low latency normally). Postgres with sync-replication is PC/EC (favors consistency normally).

### Indexing Strategies
- **B-Tree:** Default in RDBMS. Good for equality *and* range queries.
- **Hash Index:** O(1) equality lookups only. No ranges.
- **Composite Index:** Order matters! Must filter on a *left-prefix* (e.g., Index on `A, B, C` works for queries on `A` and `A, B`, but not `B` alone).
- **Geospatial:** Geohash/H3 (converts 2D to 1D strings/grids, good for Redis) vs. PostGIS/R-Tree (true polygon containment).

### Performance Bottlenecks & Fixes
- **N+1 queries:** Use DataLoaders or JOINs.
- **Connection exhaustion:** Use a pooler (PgBouncer).
- **Hot Rows:** Use optimistic concurrency or sharded counters.
- **Missing Indexes:** Look for sequential scans using `EXPLAIN ANALYZE`.

---

## Case Study: Local Delivery (Uber Eats / DoorDash)
- **Requirements:** 50 Cities, 5M orders/day, 150k active couriers. Matching latency < 200ms.
- **Estimations:** 350 orders/sec (peak) vs **37,500 location updates/sec (peak)**. This 100x difference means location data CANNOT go in the primary order database.
- **Architecture (Polyglot):**
  - **Orders & Payments (ACID / CP / EC):** Postgres, sharded by `city_id`.
  - **Courier Location (BASE / AP / EL):** Redis Geo or H3 grid in-memory. Dropping a location ping is fine; rejecting an order is not.
  - **Event Bus:** Kafka streams location events to analytics/ETA models.
- **Aha! Insights:**
  - **Separate the transactional core from the hot path.**
  - Use an in-memory geospatial store (Redis/H3) for the sub-millisecond radius search, not a disk-bound PostGIS instance.
