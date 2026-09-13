# Module 5: Caching Fundamentals

## Core Concepts

### Caching patterns

**Cache-aside (lazy loading).** The application, not the cache, owns the read path. On a read, the app checks the cache; on a miss it queries the source of truth, populates the cache, and returns the value. On a write, the app writes to the database and either deletes or updates the corresponding cache key. This is the most common pattern because it is simple, resilient to cache failure (a dead cache just means every request falls through to the DB), and only caches data that is actually requested. The consistency risk is a race between a concurrent write and a read-repopulate: a stale value can be written back into the cache if a read that started before an update finishes populating the cache after the update commits. Mitigate with short TTLs, versioned/timestamped values, or delete-then-write-with-delay strategies. Appropriate for read-heavy workloads where some staleness is tolerable — product catalogs, user profiles, event/venue metadata.

**Read-through.** Functionally similar to cache-aside, but the cache library/provider itself sits in front of the datastore and owns the miss-fill logic (e.g., via a loader function), so the application only ever talks to the cache. This centralizes the population logic (useful for consistency of loader behavior across many callers) at the cost of coupling the cache to a specific access pattern and library.

**Write-through.** Writes go to the cache first (or simultaneously), and the cache synchronously writes through to the underlying store before acknowledging. Reads are then always served from a cache that is guaranteed consistent with the DB at write time. This trades write latency (every write pays the DB write cost inline) for read consistency and simplicity — there is no invalidation logic needed since the cache is never stale immediately after a write. Good for write-then-immediately-read workloads, but doesn't help cold keys (first read of never-written data still misses), so it's often paired with cache-aside for the read-miss path.

**Write-behind (write-back).** Writes land in the cache and are acknowledged immediately; the cache asynchronously flushes to the durable store in the background (batched, coalesced, or on a timer). This gives very low write latency and can dramatically reduce DB write load (deduplicating rapid updates to the same key), but introduces a durability window — a cache node crash before flush loses writes — and complicates ordering/conflict resolution. Appropriate for high-write, loss-tolerant, or eventually-consistent use cases (view counters, activity logs, metrics aggregation); rarely appropriate for financial or inventory data.

**Consistency summary:** cache-aside and read-through are read-optimized with an inherent, boundable staleness window; write-through is consistency-optimized at write-latency cost; write-behind is latency/throughput-optimized at durability/consistency cost. None of these patterns provide strong consistency for high-contention counters — that requires bypassing the cache for the authoritative decision (see "when not to cache" below).

### Eviction policies

- **LRU (Least Recently Used):** evicts the item whose most recent access is oldest. Implemented via a hash map + doubly linked list (O(1) get/put), moving an accessed node to the head. Good default — approximates temporal locality well for most web workloads. Weak against scan-type access patterns that touch many keys once (can evict genuinely hot items) — mitigated by LRU-K or "2Q"/segmented LRU variants used by Redis (approximated LRU via sampling) and Memcached.
- **LFU (Least Frequently Used):** evicts the item with the lowest access count, better for workloads with a stable "hot set" that should survive bursts of one-off scans. Downsides: needs decay/aging (else old-but-once-popular items never get evicted — "cache pollution"), and pure counters are more expensive to maintain than a linked list. Redis's `allkeys-lfu` uses a probabilistic logarithmic counter with periodic decay to keep this cheap.
- **FIFO:** evicts the oldest-inserted item regardless of access pattern. Cheapest to implement, but ignores usage entirely, so it performs poorly whenever access frequency/recency correlates with future access (which is most real workloads). Sometimes used for simplicity in front-line CDN caches or when items have a natural, uniform lifetime.
- Practical guidance: default to LRU (or Redis's approximated LRU/LFU) unless profiling shows a scan-dominated workload (favor LFU) or a strict ordering/simplicity requirement (FIFO). Combine any of these with a TTL — eviction policy governs *capacity pressure*, TTL governs *staleness*; they solve different problems.

### Cache stampede / thundering herd

A stampede happens when a hot key expires (or the cache cluster restarts) and a large number of concurrent requests all miss simultaneously and hit the backing store at once, which can cascade into DB overload and a wider outage. Prevention patterns:

1. **Mutex/distributed locking:** on a miss, one request acquires a short-lived lock (e.g., `SET key NX PX ttl` in Redis) and recomputes the value; other requests either block briefly and retry the cache, or serve stale/default data while waiting. Requires safe lock release (Lua script comparing a token to avoid releasing someone else's lock) and a fallback if the lock holder crashes (TTL on the lock itself).
2. **Request coalescing (single-flight):** at the application layer, concurrent callers for the same key are collapsed into a single in-flight backend call, and all callers receive the same result when it resolves (Go's `singleflight`, or equivalent in-process dedup). This avoids even needing a distributed lock when misses happen concurrently within one process/fleet.
3. **Probabilistic early expiration (XFetch):** each read, before the TTL is actually reached, computes a probability of proactively refreshing based on how close the key is to expiry and how expensive the last recompute was; a small fraction of requests near expiry trigger an early, single refresh, spreading recomputation smoothly instead of a synchronized cliff-edge expiry.
4. **Stale-while-revalidate:** serve the previous (expired) value to all callers while exactly one background refresh is in flight, then swap. Bounds worst-case latency to "no worse than before" while eliminating the herd entirely, at the cost of briefly serving stale data.
5. **Jittered/staggered TTLs:** avoid setting identical TTLs on a batch of keys written at the same time (e.g., a cache warm after deploy) — add random jitter (`ttl = base ± random%`) so they don't all expire in the same instant.

### Cache invalidation taxonomy

- **TTL-based:** simplest — every entry has an expiration; staleness is bounded by the TTL. No coordination needed, but during the TTL window the value can be arbitrarily stale, and choosing a TTL is a tuning problem (too short → thrashing/thundering herd risk; too long → stale reads).
- **Event-based (write-driven) invalidation:** the writer explicitly deletes or updates the affected cache key(s) as part of the write path (in-process on cache-aside writes, or via a change-data-capture/event stream like a Debezium/Kafka pipeline for decoupled invalidation across services). Gives tighter consistency than TTL alone but requires reliably identifying every cache key derived from a piece of data — easy to miss a dependent key and leak stale data indefinitely if the event pipeline itself isn't reliable (at-least-once delivery, retries, dead-letter handling).
- **Tag-based / dependency-based invalidation:** cache entries are tagged with the entities they depend on (e.g., `event:123`, `venue:45`); invalidating a tag purges every entry carrying it. This solves the "one write, many derived cache entries" problem (e.g., an event update should invalidate the event detail page, the search index cache, and any list views containing that event) without needing to know each derived key ahead of time. Costs extra bookkeeping (a tag→keys index) and is most valuable exactly where fan-out from one entity to many cached views is otherwise unmanageable.
- In practice, production systems combine all three: TTL as a safety net (bounds worst-case staleness even if an invalidation event is dropped), event-based invalidation for the common case, and tags where a single write fans out to many cache entries.

### When NOT to cache — anti-patterns and failure modes

Caching is fundamentally an *availability/latency vs. consistency* trade. It is the wrong tool whenever the read must reflect the absolute latest state to make a correct decision, especially under contention:

- **Strongly consistent, high-contention counters/inventory** (seat availability, remaining stock, account balances, rate-limit counters): caching the *count* invites lost updates and overselling — two readers can both see "1 left," both "reserve," and both succeed. The correct pattern is to push the decision into the database (or a single-writer service) using atomic operations, conditional writes (`UPDATE ... WHERE available_count > 0`), or distributed locks/leases — and use the cache only for read-mostly *metadata* around that decision, never for the mutable, contended quantity itself.
- **Data with regulatory or financial correctness requirements** (payment status, ledger balances) — staleness here is a compliance/business risk, not just a UX blemish.
- **Write-heavy keys with low read-to-write ratio:** caching only pays off when reads significantly outnumber writes; if a key is written more than it's read, the cache invalidation/maintenance overhead exceeds the benefit.
- **Very large or rarely-repeated values:** caching a value that's essentially never read twice (a one-shot report, a unique per-request computation) wastes cache memory and evicts genuinely hot data.
- **Masking a slow/broken downstream as a "fix":** caching around a slow query treats the symptom; it also means an outage in the origin store is invisible until the cache empties or TTLs lapse en masse (which is itself a stampede risk). Fix the underlying query/index problem when possible.
- **Security-sensitive per-user data cached globally:** caching authorization decisions or personalized data under a shared key without properly namespacing by identity/tenant is a classic data-leak bug.

## Case Study Solution: Ticketmaster (Event Ticketing Platform)

### Problem statement & clarifying requirements

Design a ticket-booking platform where users browse events/venues, pick seats, hold them temporarily, and complete purchase — supporting flash-sale drops for extremely popular events (e.g., a stadium tour on-sale) without overselling a single seat.

**Functional requirements**
- Browse events, venues, and seat maps; view real-time seat availability.
- Reserve a temporary hold on one or more seats (e.g., 8–10 minutes) while the user checks out.
- Confirm purchase (payment) within the hold window; release the hold automatically on expiry or cancellation.
- Support both "best available"/general-admission (no seat map) and reserved-seating flows.

**Non-functional requirements**
- **Correctness over throughput on the critical path:** never sell the same seat twice, even under massive concurrency (no overselling).
- Handle **flash-sale spikes**: 100–1000x normal traffic in the first minutes of an on-sale.
- Low read latency for browsing (sub-100ms), acceptable added latency for the hold/purchase transaction (sub-second is fine).
- High availability for browsing/read paths even if the write path is degraded or queued.
- Auditable, idempotent purchase flow (safe retries, no duplicate charges).

### Capacity estimation

- Popular on-sale: venue capacity ~80,000 seats (stadium tour). Assume 2,000,000 fans attempt to access the sale in the first 10 minutes.
- Peak request rate to the front door: 2,000,000 users / 600s ≈ **3,300 req/s** just for page loads/queue admission, with retries and polling easily pushing observed load to **10,000+ req/s**.
- Seat-hold attempts: only admitted users reach checkout; assume the queue admits at a rate matched to checkout capacity, e.g., **50 admissions/sec**, so hold/purchase transaction rate stays bounded (~50–200 TPS) regardless of front-door load — this is the core design lever.
- Read:write ratio on browsing data (event/venue metadata, seat map layout) is extremely read-heavy: effectively **millions of reads per single write** (an event is created/updated rarely, viewed constantly) — an ideal cache-aside candidate.
- Read:write ratio on *seat inventory state* during the sale is close to 1:1 and highly contended (many readers and writers touching the same seats within seconds) — a poor caching candidate for the mutable state itself.
- Storage: 80,000 seats x ~200 bytes/row (seat id, status, price tier, version) ≈ 16MB per event — trivially small; the challenge is concurrency, not volume.

### High-level architecture

```
                          ┌───────────────────┐
      Users  ───────────▶│   CDN / Edge Cache │  (static assets, event marketing pages)
                          └─────────┬──────────┘
                                    │
                          ┌─────────▼──────────┐
                          │   API Gateway / LB  │
                          └─────────┬──────────┘
                                    │
                 ┌──────────────────┼───────────────────┐
                 ▼                                       ▼
      ┌─────────────────────┐                 ┌─────────────────────┐
      │ Virtual Waiting Room │                 │  Browse/Catalog API  │
      │ (queue, token issue)│                 │ (event, venue, seat  │
      └─────────┬───────────┘                 │  map metadata)       │
                 │ admits at controlled rate    └─────────┬───────────┘
                 ▼                                        ▼
      ┌─────────────────────┐                 ┌─────────────────────┐
      │ Reservation/Hold     │                 │  Cache-Aside Layer   │
      │ Service               │◀───────────────│  (Redis: event/venue │
      │ (holds, TTL leases)  │  read metadata  │   metadata, TTL+tags)│
      └─────────┬───────────┘                 └─────────┬───────────┘
                 │ conditional/atomic writes              │
                 ▼                                        ▼
      ┌─────────────────────┐                 ┌─────────────────────┐
      │ Seat Inventory Store  │                 │  Read Replica DB     │
      │ (source of truth,     │────replication─▶│  (catalog data)      │
      │  strong consistency,  │                 └─────────────────────┘
      │  row-level locks /    │
      │  optimistic CC)       │
      └─────────┬───────────┘
                 │ on confirm
                 ▼
      ┌─────────────────────┐        ┌─────────────────────┐
      │ Order/Payment Service │──────▶│  Message Queue (Kafka)│──▶ notifications,
      │ (idempotent, saga)    │       │  order events         │    analytics, fulfillment
      └─────────────────────┘        └─────────────────────┘
```

Key design decisions embedded in the diagram: a **virtual waiting room** throttles the effective request rate reaching the reservation/inventory path to whatever the strongly-consistent store can safely handle, decoupling front-door traffic (which can spike 100-1000x) from the transactional core (which stays roughly constant-rate). Read-heavy metadata is served from cache; the contended inventory state is not.

### API design

```
GET  /events/{eventId}                       -> event details (cacheable)
GET  /events/{eventId}/venue/seatmap         -> static seat map layout (cacheable)
GET  /events/{eventId}/seats?section=A        -> live seat availability (short-TTL/no-cache on hot sale)
POST /queue/tokens                            -> join virtual waiting room, returns queue token
GET  /queue/tokens/{token}                    -> poll queue position / admission status
POST /reservations                            -> { eventId, seatIds[], queueToken }
                                                  -> creates a time-boxed hold (idempotency key required)
                                                  returns { reservationId, expiresAt }
DELETE /reservations/{reservationId}          -> release hold early
POST /orders                                  -> { reservationId, paymentToken } (idempotency key required)
                                                  -> confirms purchase; converts hold to sold, atomically
GET  /orders/{orderId}                        -> order status
```

`POST /reservations` and `POST /orders` are idempotent (client-supplied idempotency key) so retries from flaky clients under load never double-book or double-charge.

### Data model

```
events(event_id PK, name, venue_id, start_time, status, ...)              -- read-heavy, cacheable
venues(venue_id PK, name, address, seatmap_layout_json, ...)              -- read-heavy, cacheable
seats(seat_id PK, event_id FK, section, row, seat_number, price_tier,
      status ENUM('available','held','sold'), version INT, hold_expires_at)
                                                                            -- STRONG CONSISTENCY REQUIRED
reservations(reservation_id PK, seat_ids[], user_id, status, created_at,
             expires_at)                                                  -- STRONG CONSISTENCY REQUIRED
orders(order_id PK, reservation_id FK, user_id, payment_status,
       idempotency_key UNIQUE, created_at)                                -- STRONG CONSISTENCY REQUIRED
```

`seats.status` transitions (`available → held → sold`, or `held → available` on expiry/cancel) are the linchpin of correctness: every transition must be a single atomic, conditional operation, e.g. `UPDATE seats SET status='held', version=version+1, hold_expires_at=now()+600 WHERE seat_id=? AND status='available' AND version=?` (optimistic concurrency check), or the equivalent compare-and-swap in a strongly consistent store. `events` and `venues` tolerate eventual consistency and staleness of seconds-to-minutes.

### Deep dive: caching pattern per data type, and stampede prevention

**Safe to cache (cache-aside, generous TTL, tag-based invalidation):**
- Event metadata, venue details, static seatmap geometry (SVG/coordinates): rarely change, read constantly. Cache-aside with a TTL of minutes-to-hours; invalidate via an event-driven purge (`tag:event:{id}`) whenever an admin edits the event, so the TTL is a safety net rather than the primary invalidation mechanism.
- Search/listing results and category pages: cache-aside with short TTL (seconds) plus tag invalidation on any underlying event change; CDN edge caching for the fully anonymous, non-personalized views.
- Rendered seat-map SVG/layout (shape of the venue, not live availability) — this is static per venue and cacheable indefinitely with invalidation only on venue changes.

**Must be bounded or avoided entirely (no cache-aside on the mutable value):**
- **Seat availability status** is never the *authoritative* value in a cache; it lives in the strongly consistent inventory store and every hold/purchase decision reads/writes it there via a conditional/atomic operation, not via cache-then-write. A cache is still used, but only as a **denormalized, explicitly-stale read view** for the "which seats look available" browse screen (short TTL of 1–2 seconds, clearly not used to make the reservation decision) — the actual `POST /reservations` call always re-checks and conditionally updates the source of truth, so a stale cache can only cause a *false-available* UI flicker (corrected on the failed reservation attempt), never an oversell.
- Reservation holds are similarly never cache-authoritative; the hold TTL/expiry is enforced by the inventory store itself (e.g., a `hold_expires_at` column swept by a background job, or a Redis lock used purely as an *optimization/fast-path lookup*, with the DB as the final arbiter on purchase confirmation).

**Thundering herd prevention during the flash-sale drop:**
1. **Virtual waiting room** is the primary defense: it converts an unbounded stampede of arriving users into a rate-controlled trickle of admissions, so the inventory store only ever sees a request rate it was provisioned for — this eliminates the herd before it ever reaches the cache or DB layer.
2. **Read-path stampede protection** for event/venue metadata: mutex/single-flight on cache miss (only one backend fetch per key even if thousands of requests miss simultaneously) plus jittered TTLs so cached entries for a hot event don't all expire in the same instant when the sale goes live.
3. **Write-path contention control** on the inventory store: optimistic concurrency (version column / compare-and-swap) rather than long-held pessimistic locks, so a burst of concurrent hold attempts on the same seat fails fast for all but one caller instead of queuing behind a lock; combined with sharding inventory by section/row so contention is spread across many hot rows instead of one global counter.
4. **Backpressure and queueing** between the waiting room and the inventory service (a bounded queue/rate limiter) so that even a burst that gets past admission control is smoothed before hitting the DB, and excess load degrades gracefully (users see "please wait," not 500s).

### Trade-offs and alternatives considered

- **Pessimistic locking (DB row locks) vs. optimistic concurrency control:** pessimistic locks make correctness trivial to reason about but serialize all contenders for a hot seat, hurting latency under load; OCC (version check + retry) gives better throughput under contention at the cost of retry logic in the application. Given seat-level contention is naturally sharded (each seat is only contended by however many users targeted that exact seat), OCC is the better default; a short-lived Redis lock as a fast pre-check reduces wasted DB round-trips from doomed attempts without becoming the source of truth.
- **Cache the inventory count directly for speed, and reconcile "async":** rejected — any design that lets the cache be the momentary authority on a scarce, contended resource risks overselling under exactly the load spikes the system must survive; the latency saved isn't worth the correctness risk on a purchase path.
- **Fully serialize all seat operations through a single queue/actor per event:** simpler mental model (an event-level single-writer eliminates contention races entirely) but caps a single event's throughput to one node's processing rate — acceptable for smaller events, but a poor fit for stadium-scale on-sales unless sharded per-section/per-seat-group.
- **Skip the waiting room, autoscale instead:** autoscaling the read/browse tier is fine and complementary, but the strongly-consistent inventory tier is bounded by contention (not just CPU), so autoscaling alone doesn't prevent oversell risk or DB overload; the waiting room remains the primary control even with elastic compute behind it.

### How real systems solve this

Community write-ups on Ticketmaster-style designs converge on the same shape used here: a Redis-backed **virtual waiting queue** that admits users at a controlled rate before they can even attempt a reservation; **short-TTL distributed locks** (with careful, script-guarded release) as a fast-path guard on seat holds, backed by an atomic, transactional update in the relational store as the actual source of truth; and **optimistic concurrency control** on the seat/ticket table to detect and reject conflicting concurrent reservation attempts instead of holding long pessimistic locks. Event and venue metadata is aggressively cached in Redis (`eventId → eventObject`) since it's read far more than it's written, while search uses an inverted-index engine (e.g., Elasticsearch) with its own result caching and CDN edge caching for anonymous, non-personalized queries — mirroring exactly the split described above between cacheable read-heavy metadata and the strongly-consistent, cache-avoiding inventory path.

## Sources

- [Design a Ticket Booking Site Like Ticketmaster — Hello Interview](https://www.hellointerview.com/learn/system-design/problem-breakdowns/ticketmaster)
- [Ticketmaster System Design: Building a platform that survives million-user stampedes](https://grokkingthesystemdesign.com/guides/ticketmaster-system-design/)
- [Why Ticketmaster System Design is harder than most distributed systems](https://grokkingtechcareer.substack.com/p/ticketmaster-system-design)
- [Ticketmaster System Design: Step-by-Step Guide 2026 — System Design Handbook](https://www.systemdesignhandbook.com/guides/ticketmaster-system-design/)
- [System Design Interview: Design a Ticketing System (Ticketmaster) — techinterview](https://www.techinterview.org/post/3233463451/system-design-ticketing-system-ticketmaster/)
- [Ticketmaster (Ticket Booking) System Design — System Design School](https://systemdesignschool.io/problems/ticketmaster/solution)
- [Cache Stampede Prevention — Redis Patterns (antirez)](https://redis.antirez.com/fundamental/cache-stampede-prevention.html)
- [How to Handle Cache Stampede (Thundering Herd) in Redis — OneUptime](https://oneuptime.com/blog/post/2026-01-21-redis-cache-stampede/view)
- [How to Solve the Thundering Herd Problem in Distributed Systems — Ajit Singh](https://singhajit.com/thundering-herd-problem/)
