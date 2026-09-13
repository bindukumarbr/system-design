# Module 1: System Design Thinking & Requirement Analysis

## Core Concepts

### HLD vs. LLD

**High-Level Design (HLD)** is the architecture-level view of a system: the major components (services, databases, caches, queues, load balancers), how they communicate, and the data flow between them. HLD answers "what pieces exist and how do they fit together" — it is expressed as boxes-and-arrows diagrams, not code. This is what system design interviews at the senior/staff level primarily test.

**Low-Level Design (LLD)** is the implementation-level view inside a single component: class diagrams, method signatures, database indices, concurrency handling, design patterns (e.g., factory, strategy), and pseudocode. LLD answers "how exactly does this one service work internally."

The distinction matters because conflating them is the most common interview failure mode: candidates either dive into class hierarchies before the architecture is agreed on (premature LLD), or stay so abstract ("we'll use a database") that no real engineering judgment is demonstrated (superficial HLD). A senior engineer should default to HLD, then selectively descend into LLD for the one or two components that are the crux of the problem (e.g., the ID-generation algorithm in a URL shortener, or the rate-limiting algorithm in an API gateway).

### Functional vs. Non-Functional Requirements

**Functional requirements (FRs)** describe _what_ the system does — the user-facing behaviors and features (e.g., "a user can shorten a URL," "a user can search for a driver nearby"). They define the API surface and the core use cases.

**Non-functional requirements (NFRs)** describe _how well_ the system does it — the quality attributes that shape the architecture: scalability (DAU/QPS targets), availability (99.9% vs. 99.99%), latency (p50/p99 targets), consistency model, durability, security, and cost. NFRs are usually where the real design decisions live, because two systems with identical FRs can have wildly different architectures depending on their NFRs (e.g., a URL shortener that must never lose a mapping vs. one that tolerates occasional loss).

Pitfall: candidates who list FRs exhaustively but skip NFRs entirely tend to produce designs that "work" but can't justify any technology choice, because there's no stated constraint to optimize against.

### PEDALS Framework

PEDALS (popularized by Lewis C. Lin) is a six-step interview structuring method:

- **P — Process Requirements**: Clarify the question; identify features, goals, constraints, and explicit scope (what's in/out of scope).
- **E — Estimate**: Back-of-envelope math for servers, storage, and requests-per-second at target scale (optional but strongly recommended for senior candidates).
- **D — Design the Service**: Define the key services/components needed to satisfy the functional requirements.
- **A — Articulate the Data Model**: Specify entities, tables, and fields backing each service/endpoint.
- **L — List the Architectural Components**: Name the concrete infrastructure — load balancers, caches, queues, object storage, specific cloud primitives.
- **S — Scale**: Layer in load balancing, caching, replication, sharding, and other techniques to meet the NFRs identified in step P.

PEDALS is a good default sequence for driving the interview forward linearly, especially when the interviewer wants to see incremental design (start simple, then scale).

### RESHADED Framework

RESHADED (popularized by Educative/Fahim ul Haq) is an eight-step framework, more explicit about trade-off discussion than PEDALS:

- **R — Requirements**: Full functional + non-functional requirement gathering.
- **E — Estimation**: Capacity math (traffic, storage, bandwidth, servers).
- **S — Storage Schema** (optional): Data model / schema definition.
- **H — High-Level Design**: Core building blocks and how they connect.
- **A — APIs**: Concrete interface contracts for clients to call the system.
- **D — Detailed Design**: Deep-dive into the internals of the components that matter most, addressing NFRs directly.
- **E — Evaluation**: Explicitly revisit the requirements list and show how the design satisfies each one; surface remaining trade-offs and alternatives.
- **D — Distinctive Component**: Call out the one non-obvious piece that differentiates a "generic" answer from a strong one (e.g., a custom ID-generation service, an anti-abuse pipeline).

RESHADED's advantage over PEDALS is the explicit **Evaluation** step, which forces the candidate to loop back and defend the design against the original requirements — this is precisely the moment interviewers use to gauge seniority, since junior candidates often forget to close the loop. In practice, most experienced engineers blend the two: use PEDALS' cadence to keep moving, and RESHADED's Evaluation/Distinctive-Component habit to close strong.

### Capacity Estimation Methodology

Back-of-envelope estimation converts a business-level number (DAU, or daily active users) into engineering-level numbers (QPS, storage/year, bandwidth) that justify every subsequent architecture decision. The standard chain is:

1. **DAU → daily events**: DAU × actions-per-user-per-day = total daily operations.
2. **Daily events → average QPS**: divide by 86,400 seconds/day.
3. **Average QPS → peak QPS**: multiply by a peak factor (commonly 2×–5×, since traffic isn't uniform across 24 hours).
4. **Storage**: (writes/day) × (bytes/record) × 365 × (retention years) → total storage; then add an index/replication overhead multiplier (commonly 2–3×).
5. **Bandwidth**: QPS × average payload size → ingress/egress throughput, used to size load balancers and network links.

The point of this exercise isn't precision — estimates are always order-of-magnitude — it's to reveal which resource is the actual constraint (compute, storage, or network) so the rest of the design targets the real bottleneck instead of an imagined one.

### Bottleneck Analysis & System Constraints

Every system has one or two resources that will saturate first under load — CPU, memory, disk I/O, network bandwidth, database connections, or a single-threaded coordination point (e.g., a lock, a counter, a queue partition). Bottleneck analysis means identifying that resource _before_ designing, because the mitigation differs by bottleneck type:

- **Compute-bound** → horizontal scaling behind a load balancer, stateless services.
- **Read-bound / hot-key** → caching (CDN, in-memory cache), read replicas.
- **Write-bound** → sharding, write-ahead batching, async processing via queues.
- **Storage-bound** → cold/warm/hot tiering, compression, archival to cheaper storage.
- **Coordination-bound** (e.g., a single counter for ID generation) → batching allocations per node, or moving to a coordination-free scheme (hashing).

A common pitfall is applying caching or sharding reflexively without first establishing which resource is actually under pressure — this leads to over-engineered designs that add complexity without addressing the true constraint.

### API Contract & Interface Design Principles

An API contract is the interface between the client and the system, and should be defined early (RESHADED's "A" step) because it constrains everything downstream. Good practice:

- Use resource-oriented REST semantics where reasonable (nouns, HTTP verbs, status codes) unless the domain demands RPC/streaming.
- Be explicit about idempotency (e.g., PUT vs. POST, idempotency keys for retries).
- Version the API (`/v1/...`) from day one — clients will pin to a version.
- Specify pagination, rate limits, and error shapes as part of the contract, not as an afterthought.
- Keep the contract stable even as the backend implementation evolves — this is the core value of an interface: it decouples client and server release cycles.

### Client-Server Architecture Fundamentals

The client-server model separates the requester (browser, mobile app, another service) from the responder (the backend), communicating over a network protocol (HTTP/HTTPS, gRPC, WebSockets). Key fundamentals a senior candidate should be fluent in: DNS resolution, TLS handshake overhead, load balancer placement (L4 vs. L7), stateless vs. stateful services (stateless services scale horizontally trivially; stateful ones need sticky sessions or externalized state), and the request lifecycle (client → DNS → LB → app server → cache/DB → response). Understanding where each hop adds latency is essential for meeting NFR latency targets.

### Trade-off Decision Framework: CAP Theorem & Latency vs. Consistency

**CAP theorem** states that in the presence of a network **P**artition, a distributed system must choose between **C**onsistency (every read sees the latest write) and **A**vailability (every request gets a non-error response). Partitions are a fact of life at scale, so in practice the meaningful choice is CP vs. AP during a partition — not a permanent global choice, since most real systems are tunably consistent per-operation (e.g., DynamoDB, Cassandra offer per-query consistency levels).

A closely related, often more actionable framework is **PACELC**: if there's a Partition, choose A or C; Else (normal operation), choose Latency or Consistency. This captures that even without a partition, stronger consistency (e.g., synchronous cross-region replication) costs latency. Practically: read-heavy, latency-sensitive systems (social feeds, URL redirects) lean AP/eventually-consistent with caching; systems where correctness is non-negotiable (payments, inventory counts, unique-ID allocation) lean CP even at some latency or availability cost. The trade-off framework should always be applied per-subsystem, not to the whole architecture at once — a single system commonly has both CP components (the ID/counter service) and AP components (the read/cache path).

## Case Study Solution: URL Shortener

### Problem Statement & Clarifying Requirements

**Functional requirements:**

- Given a long URL, generate a unique short URL (optionally with a user-supplied custom alias).
- Given a short URL, redirect the client to the original long URL.
- Support optional expiration dates on links.
- Support basic click-analytics (count, referrer, timestamp) — stretch goal.

**Non-functional requirements:**

- Extremely read-heavy: redirects vastly outnumber creations (commonly cited ratio ~100:1 to 1000:1).
- Low redirect latency: target p99 < 100 ms.
- High availability for redirects (99.99%) — a broken shortener breaks every link that was ever shared.
- Uniqueness of short codes must be guaranteed — a collision silently corrupting someone else's link is unacceptable.
- Short codes should be as short as possible for a given scale (aesthetics/shareability matter).
- Eventual consistency is acceptable for analytics; the URL mapping itself should be strongly consistent once created.

Scope explicitly excluded: user accounts/auth, spam/abuse detection, and a full analytics dashboard are noted as out-of-scope or stretch, per RESHADED's "define scope" discipline.

### Capacity Estimation (Applying "E" from PEDALS/RESHADED)

Assume 100M DAU-equivalent traffic profile with a 100:1 read:write ratio, and a 5-year retention window.

- **Writes (new short URLs created)**: assume 10M new links/day.
  - Average write QPS = 10,000,000 / 86,400 ≈ **116 QPS**.
  - Peak write QPS (3× factor) ≈ **350 QPS**.
- **Reads (redirects)**: 100:1 ratio → 1B redirects/day.
  - Average read QPS = 1,000,000,000 / 86,400 ≈ **11,600 QPS**.
  - Peak read QPS (3× factor) ≈ **35,000 QPS**.
- **Storage per record**: short code (7 bytes) + long URL (~500 bytes avg) + metadata (user id, timestamps, expiry, flags ≈ 100 bytes) ≈ **~600 bytes/record**.
  - 5-year total records = 10M/day × 365 × 5 ≈ **18.25 billion records**.
  - Raw storage = 18.25B × 600 bytes ≈ **~11 TB**, ×2.5 for indexes/replication ≈ **~27 TB**.
- **Bandwidth**: 35,000 QPS × ~600 bytes response ≈ **~21 MB/s egress at peak** — trivial for modern networking, confirming reads are not bandwidth-bound.

**Conclusion from estimation**: the system is overwhelmingly **read-bound** (35K QPS reads vs. 350 QPS writes), and the constraint is _not_ raw storage (27 TB fits comfortably in a horizontally-sharded or even a well-indexed single NoSQL cluster) — it is **redirect latency and read throughput at the database tier**. This single conclusion drives the entire architecture: aggressive caching in front of a modestly-sized datastore.

### High-Level Architecture

```
                     ┌───────────────┐
        write path   │   Client /    │   read path
      ┌──────────────│   Browser     │──────────────┐
      │               └───────────────┘               │
      ▼                                                ▼
┌───────────┐                                  ┌───────────────┐
│    LB /   │                                  │  CDN / Edge   │
│  API GW   │                                  │  Cache (opt)  │
└─────┬─────┘                                  └───────┬───────┘
      │                                                 │ (cache miss)
      ▼                                                 ▼
┌───────────────┐                             ┌───────────────────┐
│  Write / URL   │                             │   Redirect Service  │
│  Creation Svc  │                             │   (stateless)       │
└──────┬─────────┘                             └─────────┬──────────┘
       │  reserve batch of IDs                            │ lookup
       ▼                                                   ▼
┌───────────────┐                             ┌───────────────────┐
│  ID Generator  │                            │  Cache Layer        │
│  (Redis counter│                            │  (Redis / Memcached)│
│   or ZK/Snow-  │                            └─────────┬──────────┘
│   flake)       │                                       │ cache miss
└──────┬─────────┘                                       ▼
       │ write mapping                          ┌───────────────────┐
       ▼                                        │  Primary Datastore  │
┌───────────────────────────────────────────────│  (sharded KV/SQL)   │
│         Primary Datastore (short_code → long_url, metadata)         │
└───────────────────────────────────────────────────────────────────┘
```

**Components:**

- **API Gateway / LB**: TLS termination, routing, rate limiting per client.
- **Write/URL Creation Service**: stateless service handling `POST /urls`; validates input, requests an ID, persists the mapping.
- **ID Generator**: a counter-based service (e.g., Redis `INCR`, or a Snowflake-style distributed ID generator) that hands out unique numeric IDs, batched per app-server instance to reduce round-trips.
- **Primary Datastore**: sharded key-value store (or SQL with the short code as primary key) holding `short_code → long_url` mappings plus metadata.
- **Cache Layer**: Redis/Memcached in front of the datastore for the read path — since traffic is read-dominated and highly skewed toward recently/popularly created links, cache hit rates are typically very high.
- **CDN / Edge cache** (optional): for extremely hot links, an edge layer can serve redirects without hitting origin at all.
- **Redirect Service**: stateless service handling `GET /{short_code}`, checking cache first, falling back to the datastore, and issuing an HTTP redirect.
- **Analytics pipeline** (async, out of critical path): redirect events are fired into a queue (Kafka/Kinesis) and processed asynchronously so they never add latency to the redirect itself.

### API Design

```
POST /v1/urls
Request:
{
  "long_url": "https://example.com/some/very/long/path?query=1",
  "custom_alias": "my-link",       // optional
  "expires_at": "2027-01-01T00:00:00Z"  // optional
}
Response: 201 Created
{
  "short_url": "https://sho.rt/aZ9kLp2",
  "short_code": "aZ9kLp2",
  "long_url": "https://example.com/some/very/long/path?query=1",
  "created_at": "2026-09-05T12:00:00Z",
  "expires_at": "2027-01-01T00:00:00Z"
}
Errors: 400 (invalid URL), 409 (alias taken), 429 (rate limited)

GET /{short_code}
Response: 302 Found, Location: <long_url>
Errors: 404 (not found), 410 (expired/deleted)

GET /v1/urls/{short_code}/stats   (optional analytics endpoint)
Response: { "short_code": "...", "click_count": 12345, "last_clicked_at": "..." }
```

Design notes: `GET /{short_code}` uses **302 (Found)** rather than 301 (Moved Permanently) — this is a deliberate trade-off (see below). The API is versioned (`/v1/`) per interface-design best practice.

### Data Model

```
Table: url_mappings
  short_code      VARCHAR(10)   PRIMARY KEY
  long_url        TEXT          NOT NULL
  user_id         BIGINT        NULL (nullable for anonymous links)
  created_at      TIMESTAMP     NOT NULL
  expires_at      TIMESTAMP     NULL
  is_custom_alias BOOLEAN       DEFAULT FALSE
  status          ENUM('active','expired','deleted') DEFAULT 'active'

Table: click_events (append-only, ingested async from queue; often in a
                      columnar/analytics store rather than the primary DB)
  event_id        UUID
  short_code      VARCHAR(10)
  clicked_at      TIMESTAMP
  referrer        TEXT
  user_agent      TEXT
  ip_hash         VARCHAR(64)   -- hashed for privacy
```

`short_code` is the primary/partition key, giving O(1) lookups and natural horizontal sharding by key-range or consistent hashing.

### Deep Dive: Applying the Module's Frameworks

- **RESHADED / PEDALS applied end-to-end**: Requirements were split into FR/NFR explicitly before any component was drawn (R/P). Estimation (E) revealed the system is read-bound with a manageable storage footprint — this single number justified the caching-heavy architecture rather than an over-engineered sharded-everything design. Storage Schema (S) and API (A) were defined before Detailed Design (D) so the interface contract was stable while internals were refined. The Distinctive Component here is the **ID Generator** — it's the one piece that isn't "just add a cache/LB," and it's where a hash-based vs. counter-based decision materially changes correctness guarantees, so it gets a dedicated deep-dive (below). Evaluation (E) is the "Trade-offs" section that follows, explicitly checking the design against every stated requirement (uniqueness ✓, low-latency redirect ✓, high availability ✓, short codes ✓).
- **Bottleneck analysis applied**: because estimation showed reads at 35K QPS peak vs. writes at 350 QPS, the bottleneck is unambiguously the read path — hence cache-first architecture, read replicas, and optionally CDN, rather than investing effort in write-path sharding first.
- **CAP/PACELC trade-off applied per-subsystem**: the **ID generation and mapping-creation path is CP** — uniqueness must never be violated, so writes go through a single source of truth (or partitioned counters with disjoint ranges) even if that costs a little latency. The **redirect/read path is AP-leaning** — serving a very slightly stale cached mapping (e.g., a few seconds after an expiry update) is an acceptable trade for low latency and high availability, since a stale redirect is a minor UX issue, not a correctness violation.

### Trade-offs and Alternatives Considered

**Short code generation — hash-based vs. counter-based:**

- _Hash-based_ (e.g., MD5/SHA-256 of the long URL, truncated and base62-encoded): naturally deduplicates identical URLs, requires no central coordination, but needs collision handling (retry with salt, or a uniqueness constraint + retry) and produces less compact/predictable codes.
- _Counter-based_ (e.g., a Redis `INCR` or Snowflake-style ID, base62-encoded): guarantees no collisions by construction, produces the shortest possible codes for the current scale, but requires a centralized (or batched-per-node) coordination point. Batching — each app server reserves a block of e.g. 1,000 IDs at a time — mitigates the coordination overhead while keeping the mapping collision-free. **Chosen: counter-based with per-instance batching**, because guaranteed uniqueness without retry-loop complexity outweighs the minor loss of URL-content-based deduplication; the predictability downside (codes are enumerable/guessable) is judged acceptable since short links are meant to be publicly shared anyway, though a production system might interleave/shuffle or add a random offset to reduce trivial enumeration.

**SQL vs. NoSQL for the primary store:** Given the access pattern is a simple key lookup (`short_code → long_url`) with no complex joins or transactions, a NoSQL key-value store (e.g., DynamoDB, Cassandra) is a natural fit and scales horizontally with ease. A relational store (Postgres/MySQL) also works fine at this scale (27 TB, single key lookups) and offers stronger consistency guarantees and simpler operational tooling if the team already runs SQL infrastructure — the choice is more about team/operational familiarity than a hard technical requirement at this scale.

**301 vs. 302 redirect:** 301 (permanent) lets browsers cache the redirect, reducing origin load, but forfeits the ability to update analytics, change the destination, enforce expiration, or A/B test — since the browser may never hit the server again for that link. **302 (temporary)** is the standard choice for URL shorteners, trading a little bit of avoidable browser-cache offload for full server-side control over every redirect.

**Caching strategy:** an in-memory cache (Redis) in front of the datastore captures the read-heavy, power-law-distributed traffic (a small number of links get most of the clicks) very effectively; a CDN/edge cache adds further latency reduction for globally distributed clients at the cost of operational complexity and eventual-consistency windows on updates — a reasonable addition once traffic is geographically dispersed, but not necessary at moderate scale.

### How Real Systems Solve This

Real-world write-ups of Bitly-style systems converge on the architecture above: a stateless redirect tier backed by a cache, a datastore keyed by short code, and a counter-based (often Redis-backed, batched) ID-generation scheme to avoid hash-collision retry loops, with 302 redirects preferred over 301 specifically so operators retain control over analytics and link lifecycle. Multi-region deployments commonly assign **disjoint counter ranges per region** so ID generation never needs cross-region coordination, and CDN/edge caching is layered in for very hot links once traffic is geographically distributed, per the Hello Interview breakdown referenced below.

## Sources

- [Simplify system design interviews with the RESHADED approach — Educative](https://educative.io/blog/use-reshaded-for-system-design-interviews)
- [Intro to the PEDALS Method™ Framework for System Design — Lewis C. Lin](https://lewis-lin.com/posts/pedals-method/)
- [PEDALS™ Method: What is it? — Lewis C. Lin](https://www.lewis-lin.com/blog/pedals-method)
- [Design a URL Shortener Like Bitly — Hello Interview System Design in a Hurry](https://www.hellointerview.com/learn/system-design/problem-breakdowns/bitly)
- [Bitly System Design Explained — Educative](https://www.educative.io/blog/bitly-system-design)
- [Design a URL Shortener Like Bit.ly: A Step-by-Step Guide — System Design Handbook](https://www.systemdesignhandbook.com/guides/design-bitly/)
- [Mastering System Design Interviews with the RESHADED Approach — Vinay C.](https://www.vinayc.me/2024/01/mastering-system-design-interviews-with.html)
