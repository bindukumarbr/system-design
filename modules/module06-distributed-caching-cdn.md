# Module 6: Distributed Caching & CDN Deep Dive

## Core Concepts

### Redis Cluster: data structures, sharding, consistency

Redis is not just a key-value cache — its value is the native data structures: strings, hashes, lists, sets, sorted sets (ZSET, used constantly for feeds/leaderboards/timelines via score = timestamp), HyperLogLog (approximate cardinality), bitmaps, streams (append-only log with consumer groups, similar to a lightweight Kafka), and geospatial indexes. A ranked, paginated feed is naturally a ZSET keyed by user, scored by rank/time, retrieved with `ZREVRANGE`.

**Sharding via hash slots.** Redis Cluster splits the keyspace into a fixed **16,384 hash slots**. Every key maps to a slot via `HASH_SLOT = CRC16(key) mod 16384`, and each master node owns a contiguous subset of slots (e.g., 3 masters own roughly 5,461 slots each). Clients (with cluster-aware drivers) compute the slot locally and route directly to the owning node — there is no proxy layer, which is why Redis Cluster scales near-linearly and keeps latency low. When a key doesn't live on the node a client contacts, the node returns a `-MOVED` redirect (permanent, topology-level) or, during live resharding, a `-ASK` redirect (temporary, key-level, requires the client to send `ASKING` first). Multi-key operations (`MGET`, transactions, Lua scripts) only work when all keys hash to the **same** slot; **hash tags** (`{user1000}:profile`, `{user1000}:feed`) force related keys onto one slot by hashing only the substring between `{` and `}`, which is exactly how you'd colocate a user's feed and metadata keys for atomic multi-key ops.

**Consistency guarantees — this is the part interviewers probe.** Redis Cluster is explicitly **not** strongly consistent. It uses asynchronous primary→replica replication, so:
- A write can be acknowledged by the primary and then lost if the primary crashes before replicating to its replica (a small window, but real).
- During a network partition, a client with a stale topology view can write to a primary that a majority of the cluster has already demoted, producing a second lost-write scenario.
- Slot ownership itself converges via `configEpoch`: if two nodes claim a slot, the one with the higher epoch wins ("last failover wins") — this is eventual consistency at the topology layer, not per-key linearizability.
- A **minority partition becomes unavailable** for writes after `NODE_TIMEOUT` (fails closed); the majority partition promotes a replica and stays available after roughly `NODE_TIMEOUT` + failover time (~1-2s).

Design implication: never use Redis Cluster as a system of record needing linearizable writes (use it for that only with `WAIT`, and even then it's best-effort). Treat it as a fast, mostly-consistent cache/derived-data store, with the source of truth in a durable store (Postgres/MySQL/DynamoDB) that can rebuild cache state.

### Memcached vs Redis — when each wins architecturally

| Dimension | Memcached | Redis |
|---|---|---|
| Data model | Pure blob cache (string→bytes) | Rich structures (hash, list, ZSET, stream) |
| Concurrency model | Multi-threaded, one process fully saturates multi-core boxes for raw GET/SET | Historically single-threaded per core (I/O threading added later) — scale via sharding/cluster, not just threads |
| Persistence | None — purely volatile, restart = cold | RDB snapshot + AOF log — can survive restarts, used as a lightweight durable store |
| Replication/HA | None built-in (client-side hashing across a pool) | Built-in primary-replica, Sentinel, Cluster |
| Memory efficiency | Slab allocator, very low per-key overhead, good for huge flat caching (e.g., HTML fragment cache, session blobs) | More overhead per key given richer structures, but structure-native ops save round trips |
| Use when | You need a dumb, huge, extremely high-QPS flat cache (e.g., rendered page fragments, raw DB row cache) and want maximum throughput-per-core with zero operational complexity | You need atomic structure ops (leaderboards, counters, rate limiters, pub/sub, dedup via sets, sorted feed timelines), or need the cache to double as a lightweight durable/replicated store |

Architecturally: Memcached wins for **simple, massive, read-dominated blob caching** where you just need GET/SET at the lowest possible latency per core and don't need cross-key logic. Redis wins whenever the caching problem has **structure** — ranking, counting, deduplication, queues, or when you need built-in HA/replication instead of building it in the client. Many large systems (Facebook itself, historically) run **both**: Memcached as the giant flat look-aside cache in front of MySQL, Redis for structured derived data like feeds and counters.

### CDN cache-control semantics

- **`Cache-Control: max-age=N`** — freshness lifetime in seconds from the response's `Date`, checked by shared/private caches. **`s-maxage`** overrides `max-age` specifically for shared caches (CDN/proxy), letting you serve a page privately-fresh for 0s to the browser but cache it at the edge for 300s.
- **`no-cache`** — misleading name: caching *is* allowed, but the cache **must revalidate** with the origin (conditional GET) before serving, every time. Contrast with **`no-store`**, which forbids caching or storage entirely (used for sensitive responses).
- **`private` / `public`** — `private` restricts caching to the end-user's own browser cache, forbidding shared/CDN caches from storing it (used for personalized content); `public` explicitly allows shared caches even for responses that would otherwise be private (e.g., with auth headers present).
- **`must-revalidate`** — once stale, the cache must not serve the stale copy even under network-failure "stale-while-erroring" leniency; it must go to origin.
- **`stale-while-revalidate=N`** — serve stale for up to N seconds while asynchronously refetching in the background, hiding origin latency from the user (huge latency win for a feed's static assets/config).
- **ETag / If-None-Match** — a content fingerprint (hash or version tag) enabling conditional requests: the client sends `If-None-Match: <etag>`, and the origin returns **304 Not Modified** (no body) if unchanged, saving bandwidth while still validating freshness on every request. `Last-Modified`/`If-Modified-Since` is the coarser, timestamp-based equivalent.
- **`Vary`** — tells caches which request headers affect the response representation, so responses must be cached per distinct value of those headers. `Vary: Accept-Encoding` is nearly universal (gzip vs br vs identity bodies differ); `Vary: Authorization` or `Vary: Cookie` effectively make a response uncacheable at shared caches per-value (each cookie value becomes a different cache key) — a common accidental cause of a "CDN never hits" bug when personalization headers leak into `Vary`.

### Edge caching and latency optimization

A CDN's value is twofold: (1) moving bytes physically closer to users (speed of light is the hard floor — NY↔Tokyo round trip is ~150-200ms regardless of server speed), and (2) absorbing read traffic so origin only serves cache misses. Techniques: **anycast routing** to the nearest edge POP, **origin shielding** (one mid-tier cache in front of origin so only one edge-miss request reaches origin instead of hundreds of POPs stampeding it), **TLS session resumption/0-RTT** at the edge to cut handshake round trips, **HTTP/2 or HTTP/3 (QUIC)** to reduce head-of-line blocking, and **edge compute** (Cloudflare Workers, Lambda@Edge) to run personalization/A-B logic at the edge instead of round-tripping to origin for small decisions. For a feed product specifically, CDNs cache *static* assets (images, video segments, JS bundles, profile photos) — never the personalized feed payload itself, which must come from application-tier caches.

### Multi-tier caching (L1/L2/L3) and geo-replication

- **L1 — in-process cache** (e.g., Caffeine/Guava on the JVM, an LRU map in the app process): nanosecond-to-microsecond latency, no network hop, but capacity is bounded by one instance's memory and is **not shared** across the fleet — every instance has its own copy, so invalidation across N instances is a fan-out problem (or you accept short TTL-based staleness). Best for extremely hot, small, slow-changing data (feature flags, viewer's own profile, ranking model weights).
- **L2 — distributed cache** (Redis/Memcached cluster): shared across the fleet, sub-millisecond to low-single-digit-millisecond latency over the network, capacity scales with the cluster. This is where computed feed pages, session data, and social-graph edges live.
- **L3 — CDN edge cache**: for static/semi-static assets and, in read-heavy public APIs, cacheable API responses. Highest latency floor for a miss (must reach origin) but removes load furthest upstream and closest to the user.

The general rule: **check L1 → L2 → L3/origin**, populate every tier on the way back down (cache promotion), and set TTLs that get *shorter* as you go down the stack (L1 seconds, L2 tens of seconds to minutes, L3/CDN minutes to hours for static assets). **Geo-distributed replication**: for global products, run regional L2 clusters near each region's app tier (read-local, avoid cross-region round trips), with a designated write region and asynchronous cross-region replication (Redis's own replication, or an app-level pub/sub fan-out) — accepting eventual consistency across regions in exchange for local read latency, mirroring the same primary-region-write pattern used for the durable datastore.

### Cache warming for cold-start problems

Cold caches cause thundering-herd load spikes on origin right after a deploy, a cache-cluster failover, or for a brand-new user with no precomputed data. Strategies:
1. **Pre-warming on deploy/failover**: replay a sample of recent production traffic (or a recorded access log) against the new cache instance before it takes live traffic, or clone data from a warm replica rather than starting empty.
2. **Request coalescing / single-flight**: when many concurrent requests miss on the same key, let only one go to origin and have the rest wait on that in-flight result, instead of all N hitting origin simultaneously.
3. **Probabilistic early expiration**: recompute a soon-to-expire key slightly before it actually expires (weighted by how close to TTL), spreading recomputation load and avoiding synchronized mass expiry ("cache stampede").
4. **New-user default/synthetic seed**: for a user with no history (new signup, or a user who just came back after a long absence), serve a cheap synthetic result (e.g., globally popular/trending content) synchronously while a background job computes their real personalized result asynchronously, then swap it in.
5. **Negative caching**: cache "not found" / empty results too (with a short TTL) so repeated misses for genuinely absent data don't keep hammering origin.

## Case Study Solution: Facebook News Feed

### Problem statement & clarifying requirements

Design a system that shows each user a personalized, ranked, continuously-scrollable feed of posts from friends, groups, and pages they follow.

**Functional requirements**
- Users can create text/photo/video posts.
- Users follow/friend other users, groups, pages (a directed social graph).
- `GET /feed` returns a ranked, paginated list of posts for the requesting user.
- Feed reflects new posts with reasonably low latency (freshness) and is personally ranked (relevance), not strictly chronological.

**Non-functional requirements**
- Read-heavy by orders of magnitude: feed reads vastly outnumber post writes.
- Low read latency (p99 well under ~200ms) — feed is the primary product surface.
- High availability, eventual consistency acceptable (a friend's new post appearing 1-5s late is fine; an actual data loss is not).
- Must handle extreme fan-out skew: celebrities/pages with tens of millions of followers alongside typical users with a few hundred friends.
- Personalization/ranking freshness vs. latency is a first-class trade-off, not an afterthought.

The core tension to name explicitly in an interview: **fan-out-on-write** (push) trades write-time cost and storage for very cheap, fast reads, but breaks down for high-fan-out accounts; **fan-out-on-read** (pull) trades cheap writes for expensive, slow reads that must merge many sources at request time. Real systems use a **hybrid**.

### Capacity estimation

Rough, defensible numbers for interview purposes (order-of-magnitude, not precision):
- ~1B daily active users, each viewing feed ~5×/day → **~5B feed reads/day** ≈ ~58K reads/sec average, several-hundred-K/sec at peak.
- ~1B users produce a total of ~500M posts/day (many users post rarely) → ~6K writes/sec average, higher at peak.
- Average fan-out (friends/followers per post) ~200-300 for typical users but up to tens of millions for celebrity pages — this long tail is *why* pure fan-out-on-write is unworkable at the top of the distribution (one celebrity post triggering 50M feed-list insertions would itself create a write storm).
- Precomputed feed store: if each user's feed cache holds ~200 post-IDs × ~say 100 bytes of metadata, that's ~20KB/user × 1B users ≈ 20TB of hot feed-index data — squarely a job for a large sharded Redis/KV cluster, not something to keep in a single node.

### High-level architecture

```
                 ┌─────────────┐
 Client (app) ── │  API Gateway │
                 └──────┬──────┘
                        │
              ┌─────────▼──────────┐
              │  Feed Service       │◄──── L1 in-process cache (hot user's own feed page)
              │ (assembles/paginates)│
              └───┬─────────┬──────┘
                  │         │
     ┌────────────▼──┐   ┌──▼─────────────────┐
     │ Ranking Service│   │ L2: Distributed     │
     │ (ML scoring,   │   │ Cache (Redis Cluster)│
     │ multi-pass)    │   │ - Precomputed feed   │
     └───────┬────────┘   │   ZSETs per user     │
             │            │ - Post metadata cache │
             │            │ - Social graph edges  │
             │            └──────────┬───────────┘
             │                       │
    ┌────────▼───────────┐  ┌────────▼─────────┐
    │ Post/Graph Store     │  │ Fan-out Workers   │
    │ (sharded MySQL/       │  │ (async queue      │
    │  DynamoDB — source    │  │ consumers, push    │
    │  of truth)             │  │  new post IDs into │
    └────────────────────────┘  │  followers' ZSETs) │
                                 └────────────────────┘
                        ▲
                        │ new post event
              ┌─────────┴─────────┐
              │  Post Write Path   │
              │ (write DB, publish  │
              │  to fan-out queue)  │
              └─────────────────────┘

  L3 (CDN edge): images/video/thumbnails — never the personalized feed payload itself.
```

### API design

```
POST /v1/posts
  body: { authorId, text, mediaRefs[] }
  → 201 { postId, createdAt }

GET /v1/feed?cursor={opaque}&limit=20
  → 200 {
      items: [ { postId, authorId, rankScore, renderData... }, ... ],
      nextCursor: "<opaque base64 of (rankScore, postId, seenSet-version)>"
    }
```

Pagination uses an **opaque cursor**, not an offset — offsets are unstable against a live-updating ranked feed (new items shift positions). The cursor encodes the last-seen rank score plus a tiebreak (post ID) so `ZREVRANGEBYSCORE` can resume exactly where the client left off, and optionally a session "seen" epoch so re-ranking mid-session doesn't duplicate/skip items.

### Data model

```
Post           { postId (PK), authorId, type, mediaRefs, text, createdAt, visibility }
Edge/Graph     { userId, targetId, edgeType (friend|follow|group|page), createdAt }  -- adjacency list, indexed both directions
PrecomputedFeed (per-user ZSET in Redis): key = feed:{userId}
                 member = postId, score = rankScore (or timestamp for the simple case)
RankingFeatureCache: postId → { engagementCounts, embeddingRef, decayFactor } (Redis hash, short TTL, recomputed by streaming aggregators)
```

The durable system of record is a sharded relational/NoSQL store (Post, Edge) — the interview answer commonly cited from Meta's own TAO system is a graph-shaped read-through cache over MySQL (see Sources). The **PrecomputedFeed** is explicitly a cache/derived index, rebuildable from Post + Edge if lost.

### Deep dive

**Fan-out-on-write vs fan-out-on-read.** On write: when a normal user posts, a fan-out worker (queue-driven, e.g., Kafka/SQS consumer) looks up their follower list and pushes the new `postId` into each follower's `feed:{userId}` ZSET, capped at ~200-800 recent entries per user to bound memory. Reads become a single `ZREVRANGE` — O(log N + page size) and extremely fast. The failure mode is celebrities: pushing to 50M followers on one post is a write amplification disaster and would also mean 50M keys silently referencing content the followers may never look at (huge wasted work for content only a fraction of them will scroll to). The industry-standard **hybrid** (used conceptually the way Twitter/Facebook-style systems describe it, e.g. the Hello Interview breakdown below): flag high-follower accounts as "not precomputed"; for those, feed assembly at read time merges the user's precomputed feed with a *pull* query for recent posts from any followed high-fan-out accounts, then re-ranks the merged set. This bounds worst-case write cost to a small, well-known set of accounts while keeping the read path fast for everyone else.

**Where Redis vs Memcached fits.** The **PrecomputedFeed** and **ranking feature cache** are natural Redis ZSET/hash use cases — you need ordered structure (`ZREVRANGE`, `ZADD` with score, `ZREMRANGEBYRANK` to cap list length) that Memcached simply doesn't offer atomically. Post **content/metadata** (the actual text/media blob, immutable once written, requested by ID with no structural operations) is exactly the flat-blob, maximum-throughput case Memcached is built for — many large feed systems use Memcached as the giant look-aside cache for raw post/user objects in front of the durable store, and Redis for the structured, ranked, mutable feed index. This division (Memcached for dumb blob cache, Redis for structured derived data) mirrors the general architectural guidance above.

**Multi-tier caching for this system.** L1 (in-process, per app-server): the requesting user's own most-recently-fetched feed page, and globally-shared read-mostly data like trending/ranking-model config — nanosecond reads, avoids a network hop on pagination within one session. L2 (Redis Cluster, sharded by `userId` using hash tags like `{userId}:feed` so a user's feed ZSET and their feature cache colocate on one slot for atomic multi-key pipelines): the precomputed feed index and post feature cache, shared across the whole app fleet, sized to hold the ~20TB hot working set estimated above. L3 (CDN): every image/video/thumbnail referenced by posts — this is the highest-volume byte traffic in the whole product and is fully cacheable/immutable by content hash, so it belongs at the edge, not in application caches at all.

**Cache warming.** New user with an empty social graph and empty `feed:{userId}`: serve a synthetic "recommended/trending" feed synchronously (globally popular public content, cheap to precompute once and share across all cold-start users) while a background job walks their graph and populates their real feed asynchronously; swap over on next fetch. Returning user after a long absence: their ZSET is stale/evicted — treat it like a cold miss, rebuild via the fan-out-on-read merge path (pull recent posts from their graph), and use single-flight so concurrent app-server requests for the same stale user don't all hit the graph store simultaneously.

### Trade-offs and alternatives considered

- **Pure fan-out-on-write** — simplest reads, catastrophic write amplification for celebrities; rejected as sole strategy.
- **Pure fan-out-on-read** — no write amplification, but every feed load does an expensive multi-source merge+rank over potentially thousands of friends — too slow at p99 for a primary product surface; rejected as sole strategy.
- **Hybrid (chosen)** — bounded write cost, fast reads for the common case, added complexity of two code paths and a follower-count threshold to tune.
- **Strict consistency for feed freshness** — rejected; the product tolerates a few seconds of staleness, so async fan-out queues and Redis's eventual-consistency replication are an acceptable, much cheaper trade against strict read-your-writes guarantees (though "see your own new post immediately" is typically special-cased by reading from the write path directly).

### How real systems solve this

Meta's own infrastructure historically backs this kind of graph-shaped, read-dominated workload with **TAO**, a distributed graph-aware caching layer over sharded MySQL purpose-built for the social graph (objects and associations, i.e., posts and edges) that serves an enormous read/write ratio with tunable consistency — the same "cache reads, durable store as source of truth" split described above, just formalized as a dedicated system rather than generic Redis/Memcached. Meta's published ranking work describes an **inventory → multi-pass scoring → contextual re-ranking** pipeline (a lightweight "Pass 0" filters candidates down before expensive multitask neural-net scoring in "Pass 1," followed by diversity/contextual adjustments in "Pass 2") — the multi-stage funnel pattern (cheap filter → expensive rank on a shortlist) is directly reusable in any large-candidate-set ranking design, caching-adjacent or not. Community system-design breakdowns of this exact problem (Hello Interview, listed below) converge on the same hybrid fan-out + Redis feed-index + replicated post cache described in the deep dive above, which is why this design is treated as the standard reference answer for this interview question.

## Sources

- [News Feed ranking, powered by machine learning — Engineering at Meta](https://engineering.fb.com/2021/01/26/ml-applications/news-feed-ranking/)
- [Client-side ranking to more efficiently show people stories in feed — Engineering at Meta](https://engineering.fb.com/2016/10/20/networking-traffic/client-side-ranking-to-more-efficiently-show-people-stories-in-feed/)
- [Design Facebook's News Feed | Hello Interview System Design in a Hurry](https://www.hellointerview.com/learn/system-design/problem-breakdowns/fb-news-feed)
- [Baseline System Design — Facebook Newsfeed And "Fanout" (Medium)](https://corgicorporation.medium.com/baseline-system-design-facebook-newsfeed-and-fanout-e95311d52f65)
- [Redis Cluster Specification — Redis Docs](https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/)
- [Scale with Redis Cluster — Redis Docs](https://redis.io/docs/latest/operate/oss_and_stack/management/scaling/)
- [Hash Slot vs. Consistent Hashing in Redis — Severalnines](https://severalnines.com/blog/hash-slot-vs-consistent-hashing-redis/)
- TAO: Facebook's Distributed Data Store for the Social Graph (USENIX ATC 2013) — the canonical published reference for Meta's graph-caching architecture underlying News Feed's data layer.
