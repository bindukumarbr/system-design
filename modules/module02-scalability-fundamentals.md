# Module 2: Scalability Fundamentals

## Core Concepts

### Vertical vs. Horizontal Scaling

**Vertical scaling (scale-up)** means adding more resources (CPU, RAM, faster disks/NVMe, better NICs) to a single machine. **Horizontal scaling (scale-out)** means adding more machines and distributing load across them.

Trade-offs:

- **Cost curve**: Vertical scaling has a _superlinear_ cost curve — the largest cloud instances (e.g., AWS `u-24tb1.metal` or high-memory EC2 tiers) cost disproportionately more per unit of CPU/RAM than mid-tier instances, because high-end hardware (more sockets, more memory channels, exotic cooling) commands a premium and has a small buyer pool. Horizontal scaling has a roughly _linear_ cost curve — ten commodity nodes cost ~10x one node — but adds fixed overhead (load balancers, service discovery, cross-node coordination, network egress between nodes).
- **Ceiling**: Vertical scaling has a hard physical ceiling — there is a biggest machine you can buy. Horizontal scaling has no architectural ceiling, only diminishing returns from coordination overhead (Amdahl's Law: the serial/shared portion of the workload bounds the speedup).
- **Complexity**: Vertical scaling requires no application changes — a single-instance monolith or a relational database just gets bigger and keeps working the same way. Horizontal scaling requires the application to tolerate multiple concurrent instances: no reliance on local disk state, no in-process caches that silently diverge across nodes, idempotent operations, and a data layer that can be partitioned or replicated (sharding, read replicas, distributed consensus).
- **Failure domains**: A single large vertically-scaled machine is a single point of failure — its loss takes down 100% of capacity. Horizontal scaling spreads risk: losing one of ten nodes loses ~10% of capacity, and with redundancy (N+1 or N+2) it loses none of _availability_, only some headroom.
- **When each wins**: Vertical scaling wins for stateful single-writer systems that are hard to partition (a single primary relational database, a legacy monolith not worth re-architecting, latency-sensitive in-memory workloads like some caches) and for early-stage systems where operational simplicity beats theoretical scalability. Horizontal scaling wins for stateless request-serving tiers, systems that must survive node failure without downtime, and workloads that exceed what any single machine can hold (web-scale traffic, petabyte-scale storage).

In practice, most senior-level designs use both: scale a database vertically as far as reasonable (read replicas, bigger instances) _and_ shard/partition once vertical headroom runs out, while the stateless compute tier scales out from day one because it is nearly free to do so.

### Statelessness as the Precondition for Horizontal Scale

A server is **stateless** if no request depends on data stored only in that server's memory or local disk from a prior request. Statelessness matters because it is what makes horizontal scaling _safe_: if any request can be routed to any instance and produce a correct result, you can add or remove instances freely, and a load balancer can distribute load without regard to history.

Practical techniques to achieve statelessness:

- Push session state out of the app server into a shared store (Redis/Memcached) or into the client itself (signed JWTs, encrypted cookies).
- Avoid local file writes for anything that must survive a request or be visible to other instances — write to object storage or a database instead.
- Design idempotent APIs (e.g., idempotency keys on POST/PUT) so retries against a _different_ instance after a failover don't cause duplicate side effects.
- Keep local caches as pure, disposable optimizations — a cache miss should fall back correctly, never surface stale-only truth.

A stateless tier is also what makes auto-scaling and rolling deploys safe: instances can be killed and replaced at any time with no data loss and no "warm-up" dependency on being the same instance a client talked to before.

### Load Balancing Algorithms

A load balancer distributes incoming requests across a pool of backend instances. The three classic algorithms:

**Round-robin**: Requests are dispatched to backends in fixed rotation (1, 2, 3, 1, 2, 3, …). _Weighted round-robin_ adjusts the rotation ratio to account for heterogeneous instance capacity. Mechanics are trivial (a counter mod N), so it's cheap and predictable. It wins when backend instances are homogeneous and request costs are roughly uniform. It performs poorly when request costs vary widely (a few expensive requests can pile onto one backend by chance) or when backends have long-lived connections of uneven duration.

**Least connections**: The load balancer tracks the number of active connections per backend and routes each new request to the backend with the fewest. This requires the LB to maintain live connection-count state (more overhead than round-robin, but still O(1) with a min-heap). It wins when request durations are highly variable (e.g., some requests stream large file uploads while others are quick metadata calls) — it self-corrects for backends that are currently loaded down with long requests, which round-robin cannot do. _Weighted least connections_ combines both signals.

**IP hash (a.k.a. hash-based / consistent hashing)**: A hash function over the client's IP (or another key, like session ID or user ID) deterministically maps each client to a backend. This gives you **session affinity without a shared store** — the same client reliably lands on the same backend. It wins whenever you need the same backend to keep serving the same client (e.g., an in-memory cache warmed per-user, a WebSocket connection) without touching a session store. Its failure mode: naive modulo hashing (`hash(ip) % N`) reshuffles almost every client's mapping when N changes (a backend is added/removed), causing a cache-stampede/connection-storm. **Consistent hashing** (ring-based, or with virtual nodes) fixes this — only ~1/N of mappings change when the pool resizes. IP hashing can also be defeated by NAT (many clients behind one corporate IP overload a single backend) and by clients that roam IPs (mobile networks).

Layer distinction matters too: **L4 (transport-layer)** load balancers route on IP/port/TCP state and are fast and protocol-agnostic; **L7 (application-layer)** load balancers can inspect HTTP headers, cookies, and paths, enabling content-based routing, header-based sticky sessions, and smarter health checks, at the cost of more CPU per request.

### Sticky Sessions

**Sticky sessions** (session affinity) pin a client to a specific backend instance, usually via a cookie the LB sets or reads, or via IP hash. They exist to let application code keep session state in local memory without a shared store — simple to implement, low latency (no network hop to a session store).

Failure modes:

- **Uneven load distribution**: if one backend accumulates disproportionately many "sticky" long-lived sessions, work concentrates unevenly and round-robin/least-connections load-spreading is defeated.
- **Failure amplification**: if the pinned backend dies, every client stuck to it loses their session state simultaneously — a correlated failure instead of the graceful, spread-out degradation you get with stateless routing.
- **Scaling friction**: adding backends doesn't help already-connected clients since they're pinned to existing instances; new capacity only serves new sessions.
- **Deployment pain**: rolling deploys need to drain sticky connections carefully or users get logged out / lose in-flight state.

**Alternatives**:

- **Shared session store** (Redis, Memcached, a distributed cache): any backend can serve any request by reading session state from the shared store. This is the standard fix — it keeps the compute tier stateless while still supporting "session" semantics. Adds a network hop and a new dependency to keep highly available.
- **Client-side tokens** (JWT, signed cookies): the client carries its own session state (or a claims-based proof of identity) in every request, so no server-side session store is needed at all. Scales best (zero server-side state, no extra hop) but has trade-offs: tokens can't be cheaply revoked before expiry (mitigated with short TTLs + refresh tokens, or a revocation blocklist that reintroduces shared state), and token size adds to every request's payload.

Interview framing: sticky sessions are a shortcut around fixing statelessness, not a scaling strategy — prefer shared stores or tokens whenever you control the architecture; use sticky sessions only for cases like WebSocket/long-poll connections where the client is genuinely bound to one process's in-memory connection object.

### Auto-Scaling Fundamentals and Scaling Policies

Auto-scaling adjusts the number of running instances to match demand, prerequisite on a stateless architecture (so any instance can be added/removed safely) and a health-checked load balancer (so new instances get traffic only once ready, and dying instances get drained first).

- **Metrics-based (reactive) scaling**: policies trigger on observed metrics — CPU utilization, memory, request queue depth, requests/sec, p99 latency. Target-tracking policies (e.g., "keep average CPU at 60%") are common because they self-tune the desired instance count. Reactive scaling has inherent lag: metric collection interval + decision latency + instance boot/warm-up time (which can be 30 seconds to several minutes) mean the fix arrives after the spike has partly already caused pain. Guard against thrashing with cooldown periods and separate, more conservative scale-in thresholds than scale-out thresholds.
- **Predictive/scheduled scaling**: uses historical patterns (daily/weekly traffic cycles, known marketing events) to pre-provision capacity ahead of anticipated demand, avoiding the reactive-scaling lag entirely for foreseeable spikes. Often layered on top of reactive scaling as a floor, with reactive scaling handling the unexpected residual.
- **Scale-out vs. scale-in asymmetry**: scale-out should be fast and aggressive (users feel latency immediately); scale-in should be slow and conservative (avoid flapping, and give in-flight requests time to drain before instance termination).
- **Warm pools / pre-initialized instances**: mitigate cold-start latency for spiky workloads by keeping a small buffer of already-booted, unregistered instances ready to receive traffic instantly.

### Geographic Distribution and Multi-Region Deployment

Multi-region deployment reduces latency (serve users from the nearest region) and increases availability (a regional outage doesn't take down the whole service). Patterns:

- **Active-passive (primary + DR)**: one region serves all traffic; a standby region is kept in sync (often via async replication) and promoted on failover. Simple, but wastes standby capacity and failover has some data-loss/downtime window (RPO/RTO).
- **Active-active**: multiple regions serve live traffic simultaneously, usually behind GeoDNS or an anycast/global load balancer that routes users to the nearest healthy region. Requires solving cross-region data consistency (conflict resolution, eventual consistency, or partitioning data by region/tenant so writes rarely cross regions).
- **Data locality and replication strategy**: metadata/control-plane data often uses strongly consistent replication within a region and asynchronous cross-region replication (accepting eventual consistency) for global reads; some systems partition users to a "home region" to sidestep multi-region write conflicts entirely.
- **CDN edge layer**: for read-heavy, cacheable content (static assets, and increasingly API responses), a CDN pushes content to edge PoPs far closer to users than any origin region could be, decoupling "geographic distribution of compute" from "geographic distribution of static content."

### Cost vs. Performance Trade-offs at Scale

At scale, the cheapest architecture and the fastest architecture diverge, and system design is largely about choosing a point on that curve deliberately:

- Over-provisioning for peak load wastes money during troughs; under-provisioning risks outages during spikes — auto-scaling and spot/preemptible instances (for stateless, interruption-tolerant workloads) narrow this gap.
- Multi-region active-active buys latency and availability but multiplies infrastructure cost and operational complexity roughly by the number of regions, plus cross-region data-transfer egress fees, which are often the hidden cost center.
- Caching and CDNs trade storage/staleness for reduced compute and database load — cheap relative to scaling databases, but introduce invalidation complexity.
- Storage tiering (hot/warm/cold, e.g., S3 Standard → Infrequent Access → Glacier) trades retrieval latency for large per-GB cost savings on data accessed rarely — a pattern directly relevant to the Dropbox case study below.
- Vertical scaling of a database is often the cheapest fix at moderate scale (simpler than sharding), until you hit the ceiling of the largest available instance type, at which point horizontal partitioning becomes mandatory regardless of cost.

---

## Case Study Solution: Dropbox

### Problem Statement & Clarifying Requirements

**Functional requirements**:

- Users can upload files/folders; files sync automatically across all of a user's linked devices.
- Support very large files (multi-GB) without re-uploading the whole file on every small change.
- Detect and resolve conflicting edits made on two devices while offline.
- Support offline editing — changes queue locally and sync once connectivity returns.
- Maintain file version history and allow restoring prior versions.
- Support sharing files/folders between users with permission control.

**Non-functional requirements**:

- Durability: essentially zero acceptable data loss (files are often irreplaceable).
- Availability: sync should degrade gracefully, not block local file access, during backend outages.
- Low sync latency: changes should propagate to other devices within seconds under normal conditions.
- Massive scale: hundreds of millions of users, exabytes of stored data.
- Bandwidth efficiency: minimize data transferred, especially on metered/mobile connections.
- Security: encryption at rest and in transit; access control on shared content.

**Out of scope for this solution** (assumed handled elsewhere): billing, admin console, real-time collaborative co-editing (Google-Docs-style OT/CRDT text merging) — Dropbox historically treats whole-file/whole-block sync, not live co-editing.

### Capacity Estimation

Assume:

- 500 million registered users, 100 million daily active, averaging 2 devices each linked.
- Average stored data per user: 5 GB (highly skewed — many light users, some power users with TBs).
- Total stored data: 500M × 5 GB = **2.5 exabytes** (order-of-magnitude consistent with Dropbox's publicly known multi-exabyte Magic Pocket footprint).
- Average file change ("sync event") rate: assume each DAU triggers ~10 file-change events/day (saves, edits, new files). 100M × 10 = 1 billion events/day ≈ **~11,600 events/sec average**, with peak traffic (business hours, batch operations) at 5-10x average → **~60,000-100,000 events/sec peak**.
- Block size: Dropbox chunks files into ~4 MB blocks. A 5 GB average footprint per user across 500M users implies on the order of tens of billions of unique blocks system-wide (deduplicated further by content-hash matching across users).
- Metadata QPS: every sync event requires a metadata lookup/update plus a delta-fetch check from other linked devices — assume metadata QPS is 5-10x raw event rate → **~300,000-1,000,000 metadata ops/sec peak**, which is why the metadata layer (not block storage) is usually the harder scaling problem.

These numbers justify the architecture below: block storage must scale to exabytes with cheap, tiered redundancy, while the metadata/notification path must scale to very high QPS with low latency — two very different scaling problems solved by two very different subsystems.

### High-Level Architecture

```
                     ┌───────────────────────┐
                     │   Client Sync Agent    │  (per device: watches FS,
                     │ (chunker, diff, queue) │   chunks files, queues ops)
                     └───────────┬───────────┘
                                 │ HTTPS/gRPC (stateless API tier)
                                 ▼
                     ┌───────────────────────┐
                     │   API / Sync Gateway   │  (stateless, horizontally
                     │   (auth, routing)      │   scaled, behind L7 LB using
                     └──────┬─────────┬───────┘   least-connections)
                            │         │
              ┌─────────────┘         └─────────────┐
              ▼                                      ▼
   ┌────────────────────┐                 ┌────────────────────────┐
   │  Metadata Service    │                 │   Block Storage        │
   │ (file tree, versions,│◄───────────────►│  ("Magic Pocket"-style) │
   │  block manifests;    │  block hash refs │  content-addressed,    │
   │  sharded relational  │                 │  immutable 4MB blocks, │
   │  store)               │                 │  erasure-coded / tiered│
   └─────────┬────────────┘                 └────────────────────────┘
             │
             ▼
   ┌────────────────────┐
   │ Notification /      │  (long-lived connections / pub-sub;
   │ Presence Service     │   tells other linked devices "something
   │ (pub-sub, WebSocket  │   changed, go fetch delta" — thin signal,
   │  or long-poll)       │   not the payload itself)
   └────────────────────┘
```

Component responsibilities:

- **Client sync agent**: watches the local filesystem, splits changed files into content-addressed blocks (chunking, often with rolling hashes to isolate only the changed portion of a large file — "delta sync"), maintains a local queue of pending uploads/downloads, and reconciles conflicts.
- **API / Sync Gateway**: stateless request-handling tier; authenticates requests, routes to metadata/block services. Scales horizontally behind a load balancer.
- **Metadata service**: owns the source of truth for the file/folder tree, version history, and the list of block hashes composing each file version. This is a write-heavy, high-QPS relational workload — sharded (e.g., by user/namespace ID) since a single database cannot hold or serve this rate.
- **Block storage (Magic Pocket-style)**: content-addressed, immutable block store. Because blocks are immutable and addressed by hash, deduplication is automatic (identical block content — even across different users' files — is stored once) and blocks can be aggressively cached/CDN-fronted.
- **Notification/presence service**: a lightweight pub-sub or long-poll layer that tells a user's other linked devices "your namespace changed, go fetch the delta" without pushing the payload itself — this keeps the hot path thin and lets the sync agent decide when/how to pull.

### API Design

```
POST /api/v1/blocks/upload
  Headers: Authorization, Content-Hash: <sha256>
  Body: raw block bytes (≤4MB)
  Response: 201 { "block_hash": "...", "status": "stored" }
            or 200 { "block_hash": "...", "status": "already_exists" }  # dedup short-circuit

POST /api/v1/files/{file_id}/commit
  Body: {
    "parent_version": "<version_id>",
    "block_manifest": ["hash1", "hash2", ...],   # ordered list composing the file
    "client_mtime": "...",
    "size": 123456789
  }
  Response: 201 { "version_id": "...", "committed_at": "..." }
            or 409 { "error": "conflict", "current_version": "..." }  # optimistic concurrency

GET /api/v1/namespaces/{ns_id}/delta?cursor={cursor}
  Response: 200 {
    "entries": [ { "path": "...", "file_id": "...", "version_id": "...",
                   "deleted": false, "block_manifest": [...] }, ... ],
    "cursor": "<next_cursor>",
    "has_more": false
  }

GET /api/v1/blocks/{block_hash}
  Response: 200 <raw block bytes>  (served from cache/CDN when possible)
```

Design notes: `commit` uses the parent-version + returned 409 pattern for **optimistic concurrency control**, which is how conflicting edits are detected server-side. `delta` is cursor-paginated so a device that's been offline for a long time (or a brand-new device doing a full initial sync) can page through changes incrementally rather than pulling one giant response.

### Data Model

```
Namespace (folder/shared root)
  namespace_id (PK), owner_id, shared_with[], root_file_id

File (logical file, tracked across versions)
  file_id (PK), namespace_id (FK), path, is_deleted

FileVersion
  version_id (PK), file_id (FK), parent_version_id, size,
  block_manifest: [block_hash, ...]  (ordered),
  committed_at, committed_by_device_id, content_hash (whole-file hash)

Block
  block_hash (PK, sha256 of content), size, ref_count,
  storage_location (bucket_id / OSD reference), created_at
  # content-addressed and immutable: same hash across all users = single stored copy

Device
  device_id (PK), user_id (FK), last_sync_cursor, last_seen_at
```

`FileVersion.block_manifest` is essentially a Merkle-tree-like structure — the file's content hash is derivable from the ordered block hashes, letting the client cheaply verify integrity and letting the server cheaply detect "this block already exists" for dedup.

### Deep Dive: Applying Module 2 Concepts

- **Statelessness**: the API/sync gateway tier holds no per-user state in process memory — every request carries (or looks up) everything needed: auth token, namespace ID, cursor. This is what lets Dropbox run this tier behind a standard load balancer and auto-scale it independently of the stateful metadata/block layers.
- **Load balancing choice**: the gateway tier uses an **L7 load balancer with least-connections**, not round-robin, because request costs vary enormously — a block upload/download can hold a connection open far longer than a metadata delta-fetch — so least-connections avoids piling long transfers onto an already-busy instance. IP hashing is deliberately _not_ used here because it would create session affinity the stateless design doesn't need and would undermine even load spreading.
- **Sticky-session avoidance**: because auth and cursor state travel in the request/token rather than living in server memory, no client is pinned to a specific gateway instance — any instance can serve any request from any device, which is essential since a phone waking from sleep, a laptop reconnecting on a new network, and a desktop client can all legitimately hit different instances moment to moment.
- **Auto-scaling**: the gateway tier scales on request rate / CPU (reactive), while Dropbox's real-world traffic has predictable diurnal and weekday/weekend cycles that justify a predictive/scheduled floor on top of reactive scaling — pre-provisioning ahead of the morning "everyone opens their laptop" spike rather than reacting to it after the fact.
- **Multi-region**: the block store is geographically distributed across multiple zones/regions primarily for **durability** (a regional disaster shouldn't destroy the only copy of a block) and secondarily for **latency** (serve blocks from storage nodes close to the requesting user/region). Metadata is typically kept region-local per user's "home shard" to avoid cross-region write conflicts on the file tree, with cross-region replication for read availability and disaster recovery.

### Trade-offs and Alternatives Considered

- **Whole-file vs. block-level sync**: whole-file re-upload is simpler but wastes enormous bandwidth on small edits to large files; block-level (chunked, content-addressed) sync is more complex to implement (chunk boundary selection, manifest bookkeeping) but is essential at Dropbox's scale and is the industry-standard answer (rsync's rolling-checksum algorithm popularized this).
- **Central strong consistency vs. per-namespace sharding**: a single strongly-consistent global metadata store would be simpler to reason about but cannot scale to the QPS estimated above; sharding by namespace/user trades a small amount of cross-shard complexity (e.g., handling shared folders that span two users' shards) for near-linear metadata scalability.
- **Optimistic concurrency vs. locking**: locking a file during edit would prevent conflicts outright but is unworkable across offline devices; optimistic concurrency (detect conflict at commit time, surface a "conflicted copy" to the user) is the standard trade-off that accepts occasional user-visible conflicts in exchange for offline-first usability.
- **MySQL/relational vs. purpose-built distributed KV store** for the block index: Dropbox's own Magic Pocket team explicitly chose sharded MySQL over a novel distributed database, prioritizing operational maturity and simplicity over a theoretically more elegant but less battle-tested system — a real-world instance of "boring technology" winning at scale.

### How Real Systems Solve This

Dropbox's actual infrastructure is well-documented and matches this design closely:

- **Magic Pocket** is Dropbox's in-house exabyte-scale block storage system. It stores content as immutable, SHA-256-addressed 4MB blocks, groups them into 1GB "buckets," and replicates buckets across storage nodes called OSDs holding over 1PB each. Recently written data is multi-way replicated for immediate durability; older data is transitioned to erasure coding for storage efficiency. The system is partitioned into independent ~50PB "cells," each coordinated by a single Master, deliberately avoiding distributed-consensus complexity in favor of centralized-but-simple coordination — the team explicitly chose sharded MySQL for the block index and replication tables over a custom key-value store. (See "Inside the Magic Pocket," dropbox.tech.)
- Dropbox later **rewrote its client-side sync engine** (nicknamed "Nucleus") to unify previously divergent sync logic across platforms, improve correctness of conflict handling, and reduce sync latency — a good real-world example of how much engineering effort the _client_ side of "simple" file sync actually requires (dropbox.tech, "Rewriting the heart of our sync engine").
- Dropbox's **"Broccoli"** project ("Syncing faster by syncing less") describes concrete techniques for minimizing the data actually transferred during a sync — directly the block-level delta-sync concept covered above, applied and refined in production.
- The general industry pattern of **content-addressed, deduplicated, immutable block storage with tiered replication/erasure coding** generalizes beyond Dropbox — it's the same underlying idea in systems like Git's object store (content-addressed blobs), and cloud storage tiering (S3 Standard → Infrequent Access → Glacier) that trades retrieval latency for cost on cold data.

---

## Sources

- [Inside the Magic Pocket — Dropbox Tech Blog](https://dropbox.tech/infrastructure/inside-the-magic-pocket)
- [Rewriting the heart of our sync engine — Dropbox Tech Blog](https://dropbox.tech/infrastructure/rewriting-the-heart-of-our-sync-engine)
- [Broccoli: Syncing faster by syncing less — Dropbox Tech Blog](https://dropbox.tech/infrastructure/-broccoli--syncing-faster-by-syncing-less)
- [Testing sync at Dropbox — Dropbox Tech Blog](https://dropbox.tech/infrastructure/-testing-our-new-sync-engine)
- [How Dropbox scaled its storage infrastructure — Medium](https://medium.com/@rohitlakhotia/how-dropbox-scaled-its-storage-infrastructure-e9126970cd60)
- [Dropbox adds cold storage layer for less frequently accessed files — TechCrunch](https://techcrunch.com/2019/05/06/dropbox-adds-cold-storage-layer-for-less-frequently-access-files/)
- [Load Balancing Algorithms Explained with Code — AlgoMaster](https://blog.algomaster.io/p/load-balancing-algorithms-explained-with-code)
- [Load Balancing Strategies: L4 vs L7, Round Robin, and What "Sticky Sessions" Really Cost — Jarviix](https://jarviix.com/tech/load-balancing-strategies)
