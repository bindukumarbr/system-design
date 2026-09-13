# Module 3: Database Fundamentals

## Core Concepts

### RDBMS vs NoSQL Selection Criteria

The decision is not "SQL is old, NoSQL is web-scale" — it's a mapping from workload shape to storage engine:

| Criterion | Favor RDBMS | Favor NoSQL |
|---|---|---|
| Schema stability | Well-known, stable entities (orders, payments, ledgers) | Evolving/heterogeneous attributes (user profiles, catalogs) |
| Relationships | Many joins, multi-entity invariants (foreign keys, referential integrity) | Data naturally denormalized/hierarchical (documents, wide rows) |
| Consistency needs | Strong consistency, multi-row transactions (money movement, inventory decrement) | Eventual consistency acceptable (view counts, presence, activity feeds) |
| Scale pattern | Vertical + read replicas + sharding get you far | Horizontal write scale across many nodes, huge volume |
| Query pattern | Ad hoc queries, aggregations, reporting | Known access patterns, key-based lookups, high throughput |
| Latency profile | Tens of ms acceptable | Single-digit ms at massive QPS (session store, cache-adjacent) |

In practice, senior-level answers rarely pick one exclusively — most real systems (including the case study below) are **polyglot**: an RDBMS or NewSQL store for the transactional core, plus specialized stores (geospatial index, cache, search index, time-series store) for other access patterns.

### ACID and the BASE Model

**ACID** (traditional RDBMS transactions):
- **Atomicity** — a transaction's writes all happen or none do (all-or-nothing commit, via write-ahead log + rollback).
- **Consistency** — a transaction moves the DB from one valid state to another, respecting constraints (FKs, uniqueness, check constraints). Note: this is *application-level* consistency, distinct from the "C" in CAP.
- **Isolation** — concurrent transactions appear to execute serially from each transaction's point of view. Governed by isolation levels: Read Uncommitted → Read Committed → Repeatable Read → Serializable, each closing a specific anomaly (dirty read, non-repeatable read, phantom read) at a cost to concurrency, usually implemented via MVCC (Postgres, MySQL/InnoDB) or locking.
- **Durability** — once committed, a transaction survives crashes (fsync'd WAL, replication).

**BASE** (the NoSQL counter-model): **B**asically **A**vailable, **S**oft state, **E**ventual consistency. The system favors availability and partition tolerance, accepts that replicas may transiently disagree, and that state can change without new writes (e.g., TTL expiry, background reconciliation). BASE isn't "no guarantees" — it's a deliberate trade of strict consistency for availability and horizontal scalability, appropriate when business logic can tolerate staleness (e.g., "courier is roughly here" vs. "account balance is exactly this").

### CAP Theorem and the PACELC Extension

**CAP** (Brewer, formalized by Gilbert & Lynch): in the presence of a network **P**artition, a distributed system must choose between **C**onsistency (every read sees the latest write) and **A**vailability (every request gets a non-error response). You cannot have all three of C, A, P simultaneously when a partition actually occurs — and in any real distributed system, partitions *will* occur, so this is really a CP vs. AP choice under failure.

The common critique of CAP: it only describes behavior *during a partition*, which is rare. It says nothing about the tradeoff systems make during **normal operation**, which is where most engineering decisions actually live.

**PACELC** (Daniel Abadi, 2010) closes that gap: **if Partition, choose Availability or Consistency; Else (normal operation), choose Latency or Consistency.** This is precise and important:
- The **PAC** part is exactly CAP's partition-time tradeoff.
- The **ELC** part is new: even with *no* partition, a system replicating data must choose between returning fast from the nearest/cheapest replica (lower latency, possibly stale) or waiting for a quorum/primary acknowledgment (stronger consistency, higher latency). This tradeoff exists continuously, not just during failures.

Classifying real systems under PACELC:
- **DynamoDB, Cassandra**: PA/EL — available under partition, and by default favor low latency over strict consistency even in normal operation (tunable consistency lets you dial this).
- **MongoDB (default), traditional RDBMS with sync replication**: PC/EC — consistent under partition (refuses/blocks writes on the minority side) and consistent-favoring in normal operation (waits for replica ack).
- **Spanner/CockroachDB**: PC/EC but engineered to minimize the latency cost via synchronized clocks (TrueTime) and consensus (Paxos/Raft) — showing PACELC is a spectrum, not four buckets.

Interview framing: CAP tells you what breaks during a partition; PACELC tells you what you're already paying for in latency even when nothing is broken — which is the tradeoff that shows up on your p99 dashboards every single day.

### Database Indexing Strategies

**B-tree index** (default in Postgres, MySQL/InnoDB, SQL Server): balanced tree, O(log n) lookup, keeps keys sorted. Ideal for equality *and* range queries (`WHERE created_at BETWEEN ...`), ORDER BY, and prefix matching on composite keys. Cost: every insert/update maintains tree balance and touches multiple pages; write-heavy tables pay this cost on every secondary index.

**Hash index**: O(1) average lookup via a hash function mapping key → bucket. Only supports equality (`=`), not ranges, not sorting. Smaller and faster than B-tree for pure point lookups (e.g., Postgres hash indexes, or hash-partitioned key-value engines). Rarely the default choice in RDBMS because most workloads eventually need range queries too, and B-tree's range support at a small equality-lookup cost premium usually wins.

**Composite (multi-column) indexes**: index on `(a, b, c)` is sorted first by `a`, then `b` within `a`, then `c` within `b`. Critical rule: it serves queries filtering on a **left-prefix** of the columns (`a`, or `a,b`, or `a,b,c`) but not `b` alone or `c` alone. Column order should match: (1) equality filters first, (2) then the range filter, (3) then a column that satisfies ORDER BY, and place the highest-selectivity/most-frequently-filtered column early. Composite indexes are the main tool to make an index "covering" (all queried columns present in the index) so the query never touches the heap/table at all.

**Geospatial indexes** (relevant to the case study): a plain B-tree can't efficiently answer "find all rows within X km of this point" because lat/lng aren't naturally 1-D sortable in a way that preserves 2-D proximity. Two dominant approaches:
- **Space-filling curve encodings (Geohash / Uber's H3)**: convert 2-D coordinates into a 1-D string/integer with the property that spatially close points *usually* share a prefix, so a standard B-tree can be used to do range/prefix queries. Geohash uses interleaved lat/lng bits into a base-32 string; H3 (Uber, open source) tiles the earth into a hexagonal hierarchical grid, avoiding the geohash "boundary distortion" problem (cells near geohash grid edges can be geographically close but share no prefix). H3 is specifically built for the "find nearby drivers/orders" pattern.
- **R-tree / native geospatial types (PostGIS `GEOGRAPHY`/`GEOMETRY` + GiST index, MongoDB `2dsphere`)**: R-tree indexes group nearby objects into minimum bounding rectangles recursively, enabling true `ST_DWithin`/`$near` queries with actual great-circle distance, polygon containment, etc. More accurate and expressive than geohash prefix matching, at higher storage/maintenance cost.

In practice, high-scale dispatch systems (Uber) use a hybrid: an **in-memory hex-grid index (H3)** for the hot "who's nearby right now" query, backed by a durable geospatial-capable store for persistence and less latency-sensitive queries.

### Schema Design and Query Optimization Fundamentals

- **Normalize for correctness, denormalize for read performance.** Start from 3NF to eliminate update anomalies, then selectively denormalize (duplicate a courier's name onto the order row, maintain rollup counters) where join cost dominates a hot path.
- **Choose keys deliberately.** Surrogate integer/UUID PKs are simplest; watch out for UUIDv4 as a clustered/primary key on B-tree-organized tables — random UUIDs cause index page splits and poor locality (use UUIDv7/ULID, which are time-ordered, or a `BIGSERIAL`).
- **EXPLAIN ANALYZE everything.** Look for sequential scans on large tables, nested-loop joins against unindexed columns, and mismatched estimated-vs-actual row counts (stale statistics — run `ANALYZE`).
- **Avoid N+1 query patterns**; batch with `IN (...)`, joins, or a data-loader pattern.
- **Partition/shard large tables** by a key that matches the dominant access pattern (e.g., orders partitioned by `created_at` range for time-bounded queries and easy archival; or sharded by `customer_id`/geography for horizontal write scale).
- **Push filtering to the database**, not the application; index the columns actually used in `WHERE`, `JOIN`, and `ORDER BY`.

### Common Database Performance Bottlenecks

1. **Missing or wrong indexes** → full table scans on hot queries.
2. **Lock contention / hot rows** — e.g., decrementing the same inventory or wallet row from many concurrent transactions; mitigated with row-level optimistic concurrency, sharded counters, or queue-based serialization.
3. **N+1 queries** from ORMs lazily loading related rows.
4. **Connection pool exhaustion** — too many app instances opening direct connections; mitigated with a pooler (PgBouncer) or proxy layer.
5. **Unbounded result sets / missing pagination**, and `SELECT *` pulling unneeded columns/TOAST'd large fields.
5. **Replication lag** causing read-after-write inconsistency on read replicas.
6. **Index bloat / write amplification** from over-indexing write-heavy tables.
7. **Poor cardinality estimation / stale statistics** producing bad query plans.
8. **Cross-shard queries/joins** that require scatter-gather across many shards.

---

## Case Study Solution: Local Delivery Service

### Problem Statement & Clarifying Requirements

Design the backend for a local delivery platform (food/grocery) supporting: customers browsing merchants and placing orders, real-time matching of orders to nearby available couriers, live location tracking of couriers, and order status updates to customer and merchant.

**Functional requirements**
- Place an order (cart → checkout → order created).
- Find and assign the best available courier near the merchant.
- Courier app streams live GPS location.
- Customer/merchant can track order + courier location in real time.
- Order status state machine: `placed → accepted → preparing → courier_assigned → picked_up → in_transit → delivered / cancelled`.

**Non-functional requirements**
- Courier location updates: ~1 update every 3–5 seconds per active courier, low write latency.
- Matching decision latency: sub-second, ideally < 200ms, at metro scale.
- Order and payment data: strongly consistent (money, no double-charge, no lost order).
- Location/tracking data: availability and low latency favored over strict consistency (a few-second-old courier pin is fine).
- High read fan-out for "nearby couriers" queries during peak lunch/dinner windows.
- Regional deployment (data mostly local to a city/metro), horizontal scale across cities.

### Capacity Estimation

Assume a mid-size operator active in 50 cities:
- 5M orders/day → ~58 orders/sec average, ~5-8x peak at lunch/dinner → ~350-450 orders/sec peak.
- 500K active couriers platform-wide; ~150K online concurrently at peak, each pinging location every 4s → **~37,500 location writes/sec** at peak. This dwarfs order-write volume by ~100x — the defining capacity fact of this system, and it's why location data gets its own storage path, not the orders DB.
- Each location ping is tiny (courier_id, lat, lng, ts, heading ≈ 60 bytes) → ~2.2 MB/sec raw write throughput, trivially cheap on bandwidth but expensive on write-QPS/index-maintenance if put in the primary RDBMS.
- Matching: on each new order, query couriers within ~2–3 km of the merchant — a radius search against the ~150K online couriers, filtered down to a city/metro partition (a few thousand candidates), several times per order (retries) → tens of thousands of geo-queries/sec at peak.
- Orders table: 5M rows/day × ~1KB ≈ 5GB/day, ~1.8TB/year before archival — easily fits a sharded relational store with time-based partitioning/archival.

### High-Level Architecture

```
                     ┌────────────────┐
   Customer App ───▶ │  API Gateway   │◀─── Courier App (location pings)
                     └───────┬────────┘
              ┌──────────────┼───────────────────────┐
              ▼              ▼                        ▼
     ┌────────────────┐ ┌───────────────────┐ ┌──────────────────────┐
     │  Order Service  │ │ Courier Location   │ │ Courier-Matching     │
     │ (RDBMS: orders, │ │ Service            │ │ Service              │
     │  payments)      │ │ (writes to geo     │ │ (reads geo index,    │
     │  Postgres, ACID │ │  store + stream)   │ │  scoring, dispatch)  │
     └───────┬─────────┘ └─────────┬──────────┘ └──────────┬───────────┘
             │                     ▼                        │
             │           ┌─────────────────────┐            │
             │           │ Geospatial Index     │◀───────────┘
             │           │ (Redis geo / H3 grid │
             │           │  in-memory, backed   │
             │           │  by PostGIS/Mongo)   │
             │           └─────────────────────┘
             ▼
     ┌─────────────────┐        ┌───────────────────┐
     │ Postgres (sharded │      │ Kafka / event bus  │──▶ Notification service
     │  by city_id)      │      │ (order & location  │──▶ Analytics / ETA model
     └─────────────────┘        │  events)           │
                                 └───────────────────┘
```

The order-state RDBMS and the geospatial/location layer are deliberately separate systems with different consistency and latency needs — this split is the crux of the deep dive below.

### API Design

```
POST   /v1/orders
       { customer_id, merchant_id, items[], delivery_address }
       → { order_id, status: "placed", eta_estimate }

GET    /v1/orders/{order_id}
       → { order_id, status, courier: {id, name, lat, lng}, eta }

POST   /v1/orders/{order_id}/assign-courier      (internal, called by matching service)
       { courier_id }
       → { order_id, status: "courier_assigned" }

PATCH  /v1/orders/{order_id}/status
       { status: "picked_up" | "in_transit" | "delivered" | "cancelled" }

POST   /v1/couriers/{courier_id}/location          (high-frequency, from courier app)
       { lat, lng, heading, speed, ts }
       → 202 Accepted (fire-and-forget, no strong consistency needed)

GET    /v1/couriers/nearby?lat=&lng=&radius_km=     (internal, matching service)
       → [{ courier_id, lat, lng, distance_m }]

GET    /v1/orders/{order_id}/track                  (SSE/WebSocket stream)
       → server-pushed { lat, lng, status } updates
```

### Data Model

**orders** (Postgres, sharded by `city_id`, ACID-critical)
```
order_id        UUID (v7, time-ordered)  PK
customer_id     UUID
merchant_id     UUID
courier_id      UUID NULL
city_id         INT               -- shard key
status          ENUM
total_amount    NUMERIC(10,2)
created_at      TIMESTAMPTZ
updated_at      TIMESTAMPTZ

INDEX idx_orders_city_status_created (city_id, status, created_at)  -- composite: shard-local dashboards/queues
INDEX idx_orders_courier (courier_id) WHERE courier_id IS NOT NULL  -- partial index, courier's active orders
```

**couriers** (Postgres, master record — identity, vehicle, ratings; low write rate)
```
courier_id   UUID PK
city_id      INT
status       ENUM('offline','online_idle','online_busy')
rating       NUMERIC
```

**courier_locations** (NOT in the RDBMS — this is the key design decision) — an in-memory geospatial store, e.g. Redis with `GEOADD`/`GEOSEARCH` (geohash-based under the hood), or an H3-cell-keyed structure:
```
key: geo:city:{city_id}
member: courier_id
value: (lat, lng)     -- Redis encodes internally as a 52-bit geohash on a sorted set
TTL / staleness check: last_ping_ts, courier considered stale if > 30s old
```
For durability/analytics (not the hot path), location pings are also fanned out via Kafka into a time-series/log store for historical replay, ETA-model training, and audit.

**Indexing choice rationale**: the "find nearby couriers" query is executed at high QPS against a working set (online couriers per city) that fits in memory — this is precisely what an in-memory geospatial structure (Redis geo, or a custom H3-cell → courier-set hash map) is built for: O(log n) radius queries with sub-millisecond latency, versus a PostGIS GiST/R-tree query which is more accurate/expressive but pays disk I/O and is better suited to the durable, less latency-critical geospatial queries (merchant search by delivery zone, geofence/polygon containment for service-area checks).

### Deep Dive: Connecting the Concepts to This System

**RDBMS vs NoSQL, applied**: orders and payments are the textbook case for RDBMS — multi-row invariants (an order must reference a valid merchant and customer, a payment capture and order status must move together), strong consistency to prevent double-charge or lost orders, and moderate volume (thousands/sec, not millions). Courier location is the textbook case for a specialized non-relational store — extremely high write rate, no cross-row invariants, tolerates staleness, and the query shape (radius search) doesn't map well onto B-tree indexing at all. This is why the architecture is polyglot rather than "pick one database."

**ACID/BASE, applied**: order creation and courier assignment are wrapped in ACID transactions on Postgres — assigning a courier to an order and marking that courier `busy` must be atomic, or you get double-booked couriers. Location pings are BASE: a courier's position is basically-available, soft state (irrelevant a few seconds later), and eventually consistent across the matching service's view — acceptable because the cost of occasionally matching against a slightly-stale position (courier moved 20m) is negligible against the cost of adding transactional overhead to 37K writes/sec.

**CAP/PACELC, applied**: the location subsystem is explicitly **AP** — under a network partition between regions, you keep accepting location writes and serving (possibly stale) nearby-courier reads rather than blocking; a stale courier pin is a UX nit, a rejected write is a broken app. The order subsystem is explicitly **CP** — under partition, you'd rather reject/queue new orders in a degraded shard than risk a lost or duplicated payment. Under PACELC's "else" branch (normal operation, no partition): the location store is tuned **EL** (favor low latency — single in-memory node/cluster read, no cross-replica quorum wait) because dispatch decisions are made hundreds of times per second and a 5ms vs 50ms difference compounds; the orders store is tuned **EC** (favor consistency — synchronous replica ack before commit) because financial correctness outranks shaving milliseconds off checkout. This is the concrete version of "PACELC governs your p99 even when nothing is broken": every order write pays a small latency tax for durability guarantees that the location writes deliberately skip.

**Indexing strategy, applied**: `orders` uses composite B-tree indexes matching the dominant access patterns — `(city_id, status, created_at)` serves the merchant/ops dashboard query "active orders in this city sorted by age," and is a left-prefix match for `city_id` alone too. `courier_locations` uses geohash-based sorted-set radius queries (Redis) as the primary hot-path index because it's O(log n) and memory-resident; if the platform needed richer geospatial queries (delivery-zone polygons, "which service area is this address in") that's a separate PostGIS-backed store using GiST/R-tree indexes, queried far less frequently and tolerant of higher latency.

### Trade-offs and Alternatives Considered

- **Redis geo vs PostGIS for the hot matching path**: Redis wins on latency and throughput for point-radius queries against an in-memory working set; PostGIS wins on query expressiveness (polygons, precise great-circle distance, complex spatial joins) and durability. Using PostGIS as the source of truth and Redis as a synced hot cache captures both.
- **Geohash vs H3**: geohash is simpler and built into Redis natively; H3's hexagonal cells avoid the "two nearby points, no shared prefix" edge-of-cell problem and support elegant k-ring neighbor expansion (useful for "expand search radius if no courier found") — worth adopting once search-quality issues at geohash cell boundaries actually show up.
- **Single global Postgres vs city-sharded**: sharding by `city_id` matches the natural query boundary (almost nothing crosses cities) and lets each shard/region fail independently — a partition in one metro doesn't take down others.
- **Synchronous vs asynchronous courier assignment**: could do a synchronous "reserve and confirm" transaction per courier candidate (safer, slower, contention on hot couriers) vs. an optimistic assignment with a fast retry-on-conflict path (chosen here) — trading a small chance of a rejected assignment for much lower p99 latency under contention.

### How Real Systems Solve This

Uber's dispatch/marketplace systems use **H3**, an open-sourced hexagonal hierarchical geospatial indexing system, specifically to make "find things near this point" queries fast and to avoid geohash boundary artifacts — this is the direct real-world analog of the matching-service geo index above. Uber also describes building **real-time, user-facing analytics on geospatial data** ("Orders Near You") backed by purpose-built indexing rather than ad hoc scans of a relational store, reinforcing the pattern of separating the transactional core from the geospatial hot path. Community system-design writeups of DoorDash- and Uber-style dispatch consistently converge on the same shape used here: a relational/ACID store for orders and payments, an in-memory or specialized geospatial index for live location/matching, and an event bus decoupling location ingestion from downstream consumers (ETA models, notifications, analytics).

---

## Sources

- [PACELC Theorem — An Extension to CAP Theorem in Distributed Systems](https://medium.com/@anmol_tomer/pacelc-theorem-an-extension-to-cap-theorem-in-distributed-systems-ed2e02ce2377)
- [What is the PACELC Theorem? Definition & FAQs | ScyllaDB](https://www.scylladb.com/glossary/pacelc-theorem/)
- [The PACELC Theorem Explained: Extending CAP With Latency](https://read.thecoder.cafe/p/pacelc)
- [CAP, PACELC, ACID, BASE - Essential Concepts for an Architect's Toolkit](https://blog.bytebytego.com/p/cap-pacelc-acid-base-essential-concepts)
- [PACELC design principle — Wikipedia](https://en.wikipedia.org/wiki/PACELC_design_principle)
- [H3: Uber's Hexagonal Hierarchical Spatial Index | Uber Blog](https://www.uber.com/us/en/blog/h3/)
- [Visualizing City Cores with H3, Uber's Open Source Geospatial Indexing System](https://www.uber.com/us/en/blog/visualizing-city-cores-with-h3/)
- [GitHub - uber/h3: Hexagonal hierarchical geospatial indexing system](https://github.com/uber/h3)
- ['Orders Near You' and User-Facing Analytics on Real-Time Geospatial Data | Uber Blog](https://www.uber.com/us/en/blog/orders-near-you/)
- [Design DoorDash: A Food Delivery System Design Walkthrough](https://copilotinterview.com/blog/design-doordash)
- [Design Uber Dispatch — The Senior+ Walkthrough](https://systemdr.systemdrd.com/p/design-uber-dispatch-the-senior-walkthrough)
