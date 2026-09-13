# Module 4: Database Scaling

## Core Concepts

### Database Sharding (Horizontal Partitioning)

Sharding splits one logical dataset across multiple physical database instances so that no single node holds all the data or absorbs all the traffic. Each shard is a fully independent database (its own disk, CPU, connection pool); the application or a routing layer decides which shard owns a given row.

**Range-based partitioning.** Rows are partitioned by contiguous key ranges — e.g. `user_id 1–1M` on shard 0, `1M–2M` on shard 1. It preserves natural ordering, so range scans ("all articles from the last hour") stay cheap and single-shard. The failure mode is **hotspotting**: if keys are monotonically increasing (auto-increment IDs, timestamps), all new writes land on the newest shard while older shards sit idle. Mitigation: pick a range key that isn't monotonic, or pre-split ranges and actively rebalance.

**Hash-based partitioning.** The shard is `hash(key) % N`. This spreads writes near-uniformly and eliminates hotspots for point lookups, at the cost of range queries — "give me all articles submitted between 2 and 3pm" now fans out to every shard. The classic problem: `% N` ties the mapping to the shard count, so adding or removing a node reshuffles almost every key (see consistent hashing below).

**Directory-based partitioning.** A separate lookup service (a small, highly-replicated key→shard map) holds the authoritative assignment. This decouples placement from any formula, so you can rebalance individual keys, move a specific "hot" tenant to its own shard, or handle non-uniform data sizes — at the cost of an extra network hop and the lookup service itself becoming a critical, must-be-highly-available component (usually solved by caching the directory aggressively client-side, e.g. Vitess's vschema or Citus's shard catalog).

Most production systems combine two of these: hash the primary entity for write distribution, but keep a directory to handle exceptions (celebrity accounts, oversized tenants) and range logic within a shard for secondary access patterns.

### SQL vs NoSQL Sharding Implementations

The theory of sharding applies universally, but the *implementation* differs wildly between traditional relational databases and modern distributed stores.

**SQL Sharding (Manual & Middleware)**
Relational databases were designed to scale vertically. Splitting them horizontally breaks core RDBMS features: cross-shard JOINs become incredibly expensive (or impossible), global ACID transactions require complex two-phase commits (2PC), and foreign key constraints cannot be enforced across shards. 
- **Application-Level Sharding:** Historically, the application code itself maintained multiple database connection strings and executed the routing logic (e.g., `if user_id % 2 == 0 connect to DB_A`).
- **Middleware / Proxies:** Modern architectures use transparent proxies to make a sharded fleet look like a single logical database to the application. Examples include **Vitess** (originally built by YouTube for MySQL) and **Citus** (an extension for PostgreSQL).
- **NewSQL:** A new generation of distributed SQL databases (like **Google Spanner** and **CockroachDB**) are built from the ground up to provide native auto-sharding while maintaining global ACID guarantees (often leveraging synchronized clocks and consensus protocols like Raft/Paxos).

**NoSQL Sharding (Native Auto-Sharding)**
NoSQL databases were built for horizontal scale from day one. Because data is heavily denormalized, the lack of cross-shard JOINs is an accepted design constraint rather than a broken feature. Sharding is a native, out-of-the-box capability.
- **Peer-to-Peer Ring (e.g., Cassandra):** A masterless architecture where all nodes are equal. It uses consistent hashing with virtual nodes (vnodes). Client drivers are "topology-aware" and route queries directly to the correct node holding the data, avoiding a centralized routing bottleneck.
- **Router-based (e.g., MongoDB):** Uses a dedicated routing layer (`mongos`) and Config Servers. The Config Servers track how data is chunked and which shard owns which chunk. The `mongos` router directs queries accordingly.
- **Fully Managed (e.g., DynamoDB):** The partitioning strategy is completely abstracted from the user. DynamoDB automatically splits and redistributes data across physical partitions under the hood based on the hash of your chosen partition key and the table's provisioned throughput.

### Consistent Hashing for Shard Routing

Plain modulo hashing (`hash(key) % N`) remaps roughly `(N-1)/N` of all keys whenever N changes — adding one node to a 10-node cluster invalidates ~90% of the mapping, meaning almost the whole dataset must move. Consistent hashing fixes this.

**Algorithm.** Both shards and keys are hashed onto the same fixed circular space (e.g., a 32- or 64-bit ring, 0 to 2^32-1). Each shard occupies one or more points on the ring. A key is owned by the first shard encountered walking clockwise from the key's hash position. To add a shard, you hash it onto the ring and it only claims the slice of keys between it and its counter-clockwise neighbor — every other shard's ownership is untouched. To remove a shard, its keys simply fall to the next shard clockwise. Only `~1/N` of keys move on any single membership change, not the whole dataset.

**Virtual nodes.** Placing each physical shard at a single ring position creates uneven load if hashes cluster unevenly, and a removed node dumps its entire range onto exactly one neighbor. The standard fix is virtual nodes: each physical shard is hashed to many points on the ring (100–200 is typical), so its load is spread across many small arcs and a failure's impact is spread across many neighbors rather than concentrated on one. This is exactly how Amazon Dynamo, Cassandra, and most hash-ring based routers (e.g., Vitess-style consistent hashing) implement shard/node placement.

**Why this matters for a shard router:** the router (or client library) keeps the ring in memory, computes `hash(shard_key)`, and walks to the next node clockwise — an O(log N) operation with a sorted structure. Rebalancing on scale-out becomes a targeted, bounded data migration instead of an all-shard reshuffle, which is the property that makes online resharding operationally feasible.

### Read/Write Separation: Primary-Replica Pattern

In a primary-replica (leader-follower) topology, all writes go to a single primary, which streams a replication log (e.g., Postgres WAL, MySQL binlog) to one or more read replicas. Replicas apply the log asynchronously (usually) and serve read traffic.

**Why:** most systems are read-heavy — a news aggregator sees orders of magnitude more feed reads than submissions/votes — so replicas let you scale read capacity horizontally (add more replicas) independent of write capacity (bounded by one primary's disk/CPU). Replicas also serve as a resilience layer (promote one on primary failure) and can be placed close to users geographically to cut read latency.

**Replication lag implications.** Asynchronous replication means a replica can be milliseconds to (under load) seconds behind the primary. Consequences to design around:
- **Read-your-writes inconsistency**: a user submits a vote, then immediately re-reads the feed from a lagging replica and doesn't see it. Fixes: route the user's own subsequent reads to the primary for a short window, use a session-sticky replica, or read from whichever replica has caught up past a client-tracked LSN/sequence number.
- **Monotonic-read violations**: a user's requests round-robin across replicas with different lag, and a page refresh appears to "go back in time." Fix: sticky routing per session, or track a minimum acceptable replica offset.
- **Cascading lag under load**: if replicas fall behind during traffic spikes, naive retry/re-read logic amplifies primary load. Mitigate with lag-aware load balancers that route around a replica once its lag crosses a threshold, and by alerting on replication lag as a first-class SLO.

### Multi-Leader Replication and Conflict Resolution

Multi-leader (multi-master) replication allows writes to be accepted at more than one node — typically one leader per datacenter/region, each replicating asynchronously to the others. This trades consistency for write availability and lower write latency in multi-region deployments (no cross-region round trip to a single primary), but concurrent writes to the same record at two leaders create conflicts that single-leader systems never see.

Conflict resolution strategies:
- **Last-Write-Wins (LWW)**: attach a timestamp (or Lamport/hybrid logical clock) to each write; on conflict, keep the one with the highest timestamp, discard the other. Simple and cheap, but silently loses data — clock skew can even pick the "wrong" logical winner. Used by Cassandra and DynamoDB (as one available policy) precisely because it needs no coordination.
- **Vector clocks**: each replica tracks a per-node counter; a write's vector clock lets the system detect whether two versions are causally ordered or truly concurrent. When concurrent, both versions are kept and surfaced to the application (or user) for merging rather than arbitrarily dropped — this was the approach Amazon Dynamo used for shopping carts.
- **CRDTs (Conflict-free Replicated Data Types)**: data structures (G-counters, PN-counters, OR-Sets, LWW-registers per field) designed so that concurrent updates always merge deterministically without coordination or manual resolution. A vote/upvote counter is a textbook PN-counter use case: each replica increments its own slot, and merging is just summing per-replica slots — commutative, associative, idempotent, so it converges regardless of merge order or network partitions.

The general lesson for an interview: pick LWW when losing rare conflicting writes is acceptable and simplicity/cost matters; pick CRDTs when the field has an algebraic structure that tolerates merge (counters, sets); reach for vector clocks plus app-level merge only when conflicts are rare but must never be silently dropped (e.g., collaborative document edits).

### Connection Pooling and Database Optimization

Every DB connection costs the server real memory and a backend process/thread (Postgres: ~5–10MB and a full OS process per connection by default). Application servers scale to hundreds of concurrent request-handling threads, but a database typically tops out productively at a few hundred concurrent connections — opening one raw connection per request exhausts the DB long before it exhausts the app tier.

**Connection pooling** maintains a fixed set of warm, already-authenticated connections that requests borrow and return. Options:
- **In-process pools** (HikariCP, Go's `database/sql`, SQLAlchemy pool) — simplest, but each app instance holds its own pool, so `pool_size × instance_count` can still overwhelm the DB as you scale out horizontally.
- **External poolers** (PgBouncer, ProxySQL, RDS Proxy) sit between the fleet and the database, multiplexing thousands of client connections down to a small number of real backend connections (transaction-mode pooling: a backend connection is only held for the duration of one transaction, then returned to the pool). This is essentially mandatory once you run more than a handful of app instances against one primary.

Other standard levers for read-heavy database optimization: covering/composite indexes matched to actual query predicates and sort order (critical for feed-ranking queries with `ORDER BY score DESC LIMIT N`); a cache tier (Redis/Memcached) in front of the DB for hot reads so the database only sees cache misses; query result denormalization (precomputed counters instead of `COUNT(*)` aggregation on read); and partitioning large tables by time (e.g., monthly article partitions) so old data can be archived or dropped from the hot working set entirely.

---

## Case Study Solution: News Aggregator

### Problem Statement & Clarifying Requirements

Design a Reddit/Hacker-News-style platform that ingests articles from many external sources (RSS feeds, partner APIs, user submissions), lets users vote and comment, ranks content, and serves a personalized/ranked home feed to a very large read audience.

**Functional requirements**
- Ingest articles continuously from thousands of external sources (RSS/Atom feeds, publisher APIs) plus direct user submissions.
- Users can upvote/downvote articles and comments.
- Serve a ranked feed (hot/top/new) per category, paginated.
- Basic comment threads per article.

**Non-functional requirements**
- Extremely read-heavy: read:write ratio on the order of 1000:1 (browsing/voting vs. submitting new content).
- Low read latency (feed page < 200ms p99).
- Eventual consistency acceptable for vote counts and rankings (a vote need not be reflected instantly to every viewer).
- High availability for reads even during partial write-path degradation.
- Ingestion pipeline must tolerate slow/unreliable third-party sources without blocking the read path.

### Capacity Estimation

- 50M daily active users, average 10 feed page loads/day → 500M feed reads/day ≈ 5,800 reads/sec average, ~30,000 reads/sec at peak (5x average, typical diurnal peak factor).
- 200K new articles/day ingested from ~50K sources; 5M votes/day and 2M comments/day → write QPS ≈ (200K + 5M + 2M)/86,400 ≈ 83 writes/sec average, a few hundred/sec at peak — three to four orders of magnitude below read QPS.
- Storage: articles average 2KB metadata each → 200K/day × 2KB ≈ 400MB/day, ~150GB/year for article metadata alone; votes are tiny (a few bytes as counters, not per-user rows if using probabilistic/aggregated counting) but a durable per-user vote-audit table at 5M/day × ~50 bytes ≈ 250MB/day, ~90GB/year.
- This confirms the core design driver: reads dominate writes by ~1000:1, so nearly all scaling effort should go into the read path (replicas, caching, denormalized ranking tables), while the write path just needs to be durable and not fall over — not necessarily fast.

### High-Level Architecture

```
 [RSS/API Sources] --poll/webhook--> [Ingestion Workers] --> [Message Queue] --> [Dedup + Enrichment Service]
                                                                                        |
                                                                                        v
                                                                              [Sharded Primary DB]
                                                                             (articles, votes, comments)
                                                                                        |
                                                                        async replication (per shard)
                                                                                        v
                                                                        [Read Replicas per shard] <---- cache-miss
                                                                                        |
                                                                                        v
[Client] --> [API Gateway / LB] --> [Feed Service] --> [Redis Cache: hot feed pages / vote counters]
                        ^                     |
                        |                     v
                [Vote/Submit API] --queue--> [Ranking Service] (recomputes scores, background job)
```

- **Ingestion pipeline**: workers poll sources or receive webhooks, push raw items onto a queue (Kafka/SQS); a dedup/enrichment stage normalizes and checks for duplicate URLs (hash of canonicalized URL) before writing to the primary shard — this decouples flaky, bursty source availability from the database write path.
- **Ranking service**: runs as a background job (not inline with reads) that periodically recomputes hotness scores (Reddit-style: `log10(max(votes,1)) + sign(votes) * age_hours / constant`) and writes precomputed, sorted "feed" entries into a cache/materialized table, so a feed read is a cheap indexed lookup, not a live aggregation over votes/comments.
- **Sharded database**: articles and votes are horizontally partitioned (see Data Model below); each shard has its own read-replica set.
- **Read replicas**: absorb the ~30K reads/sec peak; the feed service reads from replicas (or the cache in front of them) almost exclusively, only falling back to the primary for read-your-own-vote consistency.

### API Design

```
GET  /v1/feed?category=tech&sort=hot&cursor=<opaque>&limit=25
     -> [{article_id, title, url, source, score, comment_count, age}], next_cursor

GET  /v1/articles/{article_id}
     -> full article + top-level comments (paginated)

POST /v1/articles                body: {url, title, source_hint}
     -> {article_id, status: "queued"|"deduped"}      # user submission

POST /v1/articles/{article_id}/vote     body: {direction: up|down|none}
     -> {article_id, new_user_vote}                    # score updates async

POST /v1/articles/{article_id}/comments  body: {parent_id?, body}
     -> {comment_id}
```

`GET /v1/feed` is the dominant call by ~1000:1 and is the one endpoint that must be servable entirely from cache/replicas with no primary involvement. Votes and submissions are accepted immediately (durability), then processed asynchronously by the ranking pipeline — the API returns before the global score is recomputed.

### Data Model

```sql
articles (article_id PK, url_hash UNIQUE, title, source_id, submitted_at,
          upvotes, downvotes, comment_count, hot_score, category)

votes (article_id, user_id, direction, voted_at)   -- PK (article_id, user_id)

comments (comment_id PK, article_id, parent_id, user_id, body, created_at)

sources (source_id PK, feed_url, name, poll_interval)
```

**Shard key: `article_id` (hash-partitioned).** Reasoning:
- Sharding by `source_id` was considered and rejected: sources are extremely skewed (a handful of major publishers generate a large fraction of volume), which would recreate exactly the hotspot problem range/list-based partitioning is prone to — a few shards would take disproportionate write and read load.
- Sharding by `article_id` with a hash function spreads both write load (new articles) and read load (feed lookups, vote writes, comment writes — all keyed by article) roughly uniformly across shards, since article IDs are assigned independently of source popularity.
- The vote and comment tables are co-located (same shard) with their parent article by using `article_id` as the sharding/routing key for all three tables, so a single article's full state (metadata + votes + comments) lives on one shard — this makes the common operations ("get article + its votes + its comments") single-shard, avoiding distributed joins or scatter-gather for the hot path.
- The cost: building a global "top articles across all categories" feed requires a scatter-gather across shards. This is solved by *not* computing that live — the ranking service pre-aggregates a bounded top-N per shard periodically and merges/re-sorts those bounded sets (a few hundred candidates per shard, not the full dataset), which is cheap relative to a live cross-shard query.

### Deep Dive: Connecting Sharding, Consistent Hashing, Replicas, and Pooling to the Read-Heavy Pattern

1. **Sharding strategy → write scalability.** Hash-partitioning `article_id` across shards means write throughput for new articles/votes/comments scales linearly by adding shards, and no single source's popularity can overload one shard — directly addressing the ingestion pipeline's uneven, bursty nature.

2. **Consistent hashing → low-disruption growth.** As the platform grows (more sources, more traffic), shards need to be added. Routing via a consistent-hash ring (with virtual nodes per physical shard) means adding a shard only requires migrating `~1/N` of articles — the enrichment/ingestion service and the feed service both consult the same ring to route `article_id` to the correct shard, and a resharding event never requires a full-dataset rewrite or a maintenance-window stop-the-world migration. This is what makes it operationally realistic to grow shard count as content volume grows over years, not just once at launch.

3. **Read replica topology → absorbing the 1000:1 read skew.** Given peak load of ~30K reads/sec against ~a few hundred writes/sec, each shard's primary handles only its slice of the small write volume, while each shard is fronted by 3–5 read replicas (more in the busiest categories) that absorb virtually all `GET /v1/feed` and `GET /v1/articles/{id}` traffic. Because feed content and vote tallies tolerate staleness (nobody notices a score being 2 seconds old), the design leans fully into asynchronous replication and accepts replication lag — an explicit trade users don't perceive, in exchange for read capacity that scales by simply adding replicas. The one place lag matters — a user immediately re-reading their own just-cast vote — is handled narrowly: route that one confirmation read to the primary or to a replica whose replayed LSN is checked against the write's LSN, rather than paying stronger consistency everywhere.

4. **Connection pooling → making replica fan-out affordable.** With potentially hundreds of stateless feed-service instances each wanting DB access across many shard-replica pairs, raw per-request connections would multiply into tens of thousands of backend connections per shard, well past what Postgres/MySQL can hold. A pooler (e.g., PgBouncer in transaction mode) sits in front of each shard's primary and each replica, multiplexing the fleet down to a bounded number of real backend connections — this is what actually lets "add more app instances to handle read QPS" scale without the database becoming the bottleneck it was supposed to be scaled away from.

Together: consistent hashing decides *which* shard a read/write targets and lets that mapping evolve cheaply; the shard key ensures load is even in the first place; read replicas turn each shard into a read-scalable unit; and pooling is the plumbing that makes fan-out to many replicas across many app instances actually work at the connection-count level.

### Trade-offs and Alternatives Considered

- **Precomputed hot-score table vs. live scoring**: precomputing trades some staleness (scores update every N seconds rather than instantly) for making reads O(1) indexed lookups — the correct trade for a 1000:1 read-skew system.
- **Vote counters as aggregated PN-CRDT counters vs. row-per-vote with COUNT**: aggregated counters (merge-friendly, cheap to read) were chosen over live `COUNT(*)` on the votes table; the per-user vote row is kept only for idempotency (prevent double-voting) and audit, not for computing the displayed score.
- **Multi-leader replication was considered for global write availability** (writes accepted in US/EU/APAC regions independently) but rejected for the core article/vote data given the low write volume — a single-region primary per shard is simpler and write volume doesn't justify multi-leader's conflict-resolution complexity; CRDNs/CDN-level caching solve the geographic read-latency problem instead, which is the actually large workload.
- **Directory-based sharding was considered** to allow manually rebalancing a single viral article's shard under load, but was deferred in favor of pure consistent hashing plus a per-article cache — a single hot article is absorbed by caching its feed/comment reads, not by moving data.

### How Real Systems Solve This

- Reddit's own engineering writeups on their architecture and past scaling incidents describe heavy reliance on a caching tier and read-scaled data stores in front of relational storage, and their public ranking write-ups explain the hot-score formula (`log10` of vote magnitude combined with a time-decay term) that this case study's ranking service reuses directly (Amir Salihefendic, ["How Reddit ranking algorithms work"](https://medium.com/hacking-and-gonzo/how-reddit-ranking-algorithms-work-ef111e33d0d9); background on Reddit's 2010 architecture/scaling discussion on [Hacker News](https://news.ycombinator.com/item?id=1159783)).
- Hacker News's ranking algorithm (gravity-based decay: `(votes-1)^0.8 / (age+2)^gravity`) is the other canonical reference implementation for this kind of time-decayed popularity score, documented in detail in ["How Hacker News ranking algorithm works"](https://medium.com/hacking-and-gonzo/how-hacker-news-ranking-algorithm-works-1d9b0cf2c08d).
- Consistent hashing as the standard mechanism for minimal-disruption shard rebalancing (used in this design's shard router) is the same technique underlying Amazon Dynamo, Cassandra, and modern sharding middleware like Vitess; see the practical walkthroughs at [Hello Interview's consistent hashing writeup](https://www.hellointerview.com/learn/system-design/core-concepts/consistent-hashing) and [Nikki Siapno's explainer](https://blog.levelupcoding.com/p/consistent-hashing-clearly-explained).
- Range vs. hash vs. directory sharding trade-offs, and the hotspotting failure mode that motivated this design's choice of `article_id` hashing over `source_id` partitioning, are covered in [engineeringatscale's sharding strategies deep dive](https://engineeringatscale.substack.com/p/system-design-concepts-dive-deep) and [Bhagwati Malav's "From Hot Keys to Rebalancing"](https://medium.com/startlovingyourself/from-hot-keys-to-rebalancing-a-deep-dive-into-sharding-dcb48c69bab7).

## Sources

- [How Reddit ranking algorithms work — Amir Salihefendic](https://medium.com/hacking-and-gonzo/how-reddit-ranking-algorithms-work-ef111e33d0d9)
- [How Hacker News ranking algorithm works](https://medium.com/hacking-and-gonzo/how-hacker-news-ranking-algorithm-works-1d9b0cf2c08d)
- [Reddit architecture/scaling discussion — Hacker News thread](https://news.ycombinator.com/item?id=1159783)
- [Consistent Hashing for System Design Interviews — Hello Interview](https://www.hellointerview.com/learn/system-design/core-concepts/consistent-hashing)
- [Consistent Hashing Clearly Explained — Nikki Siapno](https://blog.levelupcoding.com/p/consistent-hashing-clearly-explained)
- [System Design Concepts: Database Sharding Strategies — engineeringatscale](https://engineeringatscale.substack.com/p/system-design-concepts-dive-deep)
- [From Hot Keys to Rebalancing: A Deep Dive into Sharding — Bhagwati Malav](https://medium.com/startlovingyourself/from-hot-keys-to-rebalancing-a-deep-dive-into-sharding-dcb48c69bab7)
- [Sharding, Rebalancing, and Consistent Hashing — Xavier Fang](https://medium.com/@profxfang/sharding-rebalancing-and-consistent-hashing-system-design-interviews-4b2b4cfe5fbd)
