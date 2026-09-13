# Module 03: Database Fundamentals

## Core Concepts

### RDBMS vs NoSQL

- **RDBMS (SQL):** Use for strong consistency, stable schema, multi-row ACID transactions (payments, inventory).
- **NoSQL:** Use for massive horizontal write scale, eventual consistency, evolving schemas (user profiles, catalogs).
- _Insight:_ Most real architectures are **polyglot** (e.g., SQL for payments + Redis for caching + ElasticSearch for text search).

### ACID vs BASE

- **ACID (SQL):** Atomicity, Consistency, Isolation (via MVCC/locks), Durability.
- **BASE (NoSQL):** Basically Available, Soft state, Eventual consistency. Good when business logic tolerates staleness (e.g., view counts, presence).

### PACELC Theorem (Extends CAP)

- **CAP:** If Partition, pick Availability (AP) or Consistency (CP).
- **PACELC:** Even without a partition, you must trade Latency (L) vs Consistency (C).
- _Example:_ DynamoDB defaults to PA/EL (favors low latency normally). Postgres with sync-replication is PC/EC (favors consistency normally).

### Indexing Strategies

- **B-Tree:** Default in RDBMS. Good for equality _and_ range queries.
- **Hash Index:** O(1) equality lookups only. No ranges.
- **Composite Index:** Order matters! Must filter on a _left-prefix_ (e.g., Index on `A, B, C` works for queries on `A` and `A, B`, but not `B` alone).
- **Geospatial:** Geohash/H3 (converts 2D to 1D strings/grids, good for Redis) vs. PostGIS/R-Tree (true polygon containment).

### Performance Bottlenecks & Fixes

- **N+1 queries:** Use DataLoaders or JOINs.
- **Connection exhaustion:** Use a pooler (PgBouncer).
- **Hot Rows:** Use optimistic concurrency or sharded counters.
- **Missing Indexes:** Look for sequential scans using `EXPLAIN ANALYZE`.

### Detailed Database Engine Comparison

| Engine / Type                               | Storage Underlying       | Scale Model                                                       | Consistency (PACELC)                    | Primary Use Cases                                                                     | When NOT to use (Anti-Pattern)                                                        |
| ------------------------------------------- | ------------------------ | ----------------------------------------------------------------- | --------------------------------------- | ------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| **PostgreSQL**<br/>_(Relational / SQL)_     | B-Tree (default)         | Vertical + Read Replicas (Horizontal possible via Sharding/Citus) | PC/EC (Strong Consistency)              | Financials, ledgers, complex transactions, structured data with multi-row invariants. | Massive write-throughput > 10k/sec; highly unstructured, constantly changing schemas. |
| **MySQL**<br/>_(Relational / SQL)_          | B-Tree (InnoDB)          | Vertical + Read Replicas                                          | PC/EC (Strong Consistency)              | Traditional CRUD apps, e-commerce, content management.                                | Graph relationships, pure time-series, or append-heavy logging.                       |
| **Cassandra**<br/>_(Wide-Column / NoSQL)_   | LSM-Tree                 | Native Horizontal (Peer-to-Peer Ring)                             | PA/EL (Tunable Consistency)             | Time-series, IoT, logging, high-velocity append-only writes.                          | Complex joins, ad-hoc query capabilities, strong ACID requirements.                   |
| **MongoDB**<br/>_(Document / NoSQL)_        | B-Tree (WiredTiger)      | Native Horizontal (Auto-Sharding)                                 | PA/EC (Strong by default at primary)    | Content catalogs, user profiles, gaming states, JSON/heterogeneous data.              | Highly relational data requiring deep multi-collection JOINs.                         |
| **Redis**<br/>_(Key-Value / In-Memory)_     | Hash Tables / Skip Lists | Horizontal (Redis Cluster)                                        | PA/EL (Eventual via async replication)  | Caching, rate limiting, leaderboards (Sorted Sets), ephemeral pub/sub.                | Durable, persistent source-of-truth storage for complex data (without AOF tuning).    |
| **DynamoDB**<br/>_(Key-Value / Document)_   | LSM-Tree / B-Tree hybrid | Native Horizontal (Managed)                                       | PA/EL (Eventual default, Strong opt-in) | Serverless architectures, shopping carts, session stores, fast tier-0 lookups.        | Analytics (OLAP), ad-hoc querying, complex relational graph data.                     |
| **Elasticsearch**<br/>_(Search / Document)_ | Inverted Index (Lucene)  | Native Horizontal (Sharded Indices)                               | PA/EL (Eventual Consistency)            | Full-text search, log aggregation, complex multi-parameter filtering.                 | Primary transactional datastore; high-frequency single-record updates.                |
| **Neo4j**<br/>_(Graph / NoSQL)_             | Index-free adjacency     | Vertical (Read clustering available)                              | PC/EC (ACID compliant)                  | Fraud detection, recommendation engines, social networks, complex relationships.      | Simple key-value lookups; massive horizontal write-scale logging.                     |

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
