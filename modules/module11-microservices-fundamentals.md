# Module 11: Microservices Fundamentals

## Core Concepts

### Monolith vs Microservices — When to Break Apart (and When Not To)

A monolith is a single deployable unit containing all business logic behind one process boundary, typically one codebase and one (often shared) database. Microservices decompose that unit into independently deployable services, each owning a slice of business capability and its own data store.

**Signals it's time to split:**
- **Team scaling friction** — multiple teams stepping on the same codebase, merge conflicts, release trains blocked by unrelated work (Conway's Law: org structure should mirror desired architecture, not fight it).
- **Divergent scaling profiles** — one subsystem (e.g., GPS ingestion) needs 100x the compute of another (e.g., user settings); coupling them forces over-provisioning the whole monolith.
- **Divergent release cadence** — a fast-iterating feature (social feed ranking) is blocked by the release discipline required for a slow-moving core (billing, auth).
- **Blast radius reduction** — a bug in one bounded context (image thumbnailing) should not take down checkout or auth.
- **Independent technology needs** — e.g., a geospatial matching engine benefits from a different language/runtime (Go/Rust with R-tree indexes) than the main Rails/Django app.

**When NOT to split (the more common mistake):**
- Team is small (<2 pizza teams) — the coordination overhead of a distributed system (network calls, partial failure, eventual consistency, distributed tracing, service versioning) usually costs more than it saves.
- Domain boundaries are still unclear — splitting prematurely calcifies the wrong seams, and refactoring across service/network boundaries is far more expensive than refactoring inside a single process.
- No operational maturity yet — microservices require CI/CD per service, service mesh or comparable observability, on-call rotation per service, contract testing. Without this, you get a "distributed monolith": all the coupling of a monolith plus all the latency and failure modes of a distributed system.
- The standard industry advice (popularized by Martin Fowler and confirmed by numerous postmortems) is: **start with a well-modularized monolith** (modules with clear internal boundaries, one per bounded context), and extract services only once a concrete pain point (scaling, team ownership, deployment independence) justifies the operational cost. This is the "monolith-first" strategy.

### Bounded Contexts and Domain-Driven Design Basics

Domain-Driven Design (DDD), from Eric Evans' work, is the primary discipline for finding correct service boundaries.

- **Domain**: the problem space (e.g., "fitness activity tracking").
- **Subdomain**: a partition of the domain — *core* (the differentiating capability, e.g., segment matching/leaderboards for Strava), *supporting* (necessary but not differentiating, e.g., activity upload/storage), and *generic* (solved problems you should buy/reuse, e.g., auth, notifications, payments).
- **Bounded Context**: a boundary within which a specific domain model and its ubiquitous language are consistent and unambiguous. The same real-world noun can mean different things in different contexts — e.g., "Activity" in the *Ingestion* context is a raw GPS stream; in the *Social Feed* context it's a post with kudos/comments; in the *Segment* context it's a set of effort candidates to match against polylines. DDD says: **don't force one canonical "Activity" model across all contexts** — model each context's own view and translate at the boundary (an Anti-Corruption Layer).
- **Aggregate**: a cluster of entities/value objects treated as a single consistency unit with one root entity (e.g., a `LeaderboardEntry` aggregate enforcing "only the best effort per athlete per segment is retained").
- In microservices, **one bounded context ≈ one (or a small group of) services**. Bounded contexts become your primary decomposition axis instead of technical layers (don't create a "database service," a "validation service" — these are accidental complexity, not domain boundaries).
- **Context mapping** describes how contexts relate: *Shared Kernel*, *Customer-Supplier*, *Conformist*, *Anti-Corruption Layer*, *Open Host Service* (a well-published API a context exposes to others) — these patterns describe integration contracts between the services you draw from bounded contexts.

### Service Communication Patterns

**Synchronous (REST/gRPC):**
- Used when the caller needs an immediate answer to proceed (e.g., "fetch this user's profile to render a page," "validate this token").
- REST/JSON: ubiquitous, human-debuggable, loosely coupled via HTTP semantics, but higher serialization overhead and weaker contracts (unless paired with OpenAPI).
- gRPC: HTTP/2 + Protobuf — strongly typed contracts, code-gen clients/servers, bidirectional streaming, lower latency/payload size; preferred for internal service-to-service calls at scale. Cost: less human-readable, requires schema registry/versioning discipline, less browser-native (needs grpc-web/gateway for external clients).
- Risk: synchronous call chains create **temporal coupling** — if service B is down or slow, every synchronous caller of B degrades too (cascading failure). Mitigate with timeouts, circuit breakers (e.g., Hystrix/resilience4j patterns), bulkheads, and retries with backoff + jitter.

**Asynchronous (queue/event-based):**
- Used when the caller does not need an immediate result, when you want to decouple producer/consumer lifecycles, or when one event fans out to multiple independent consumers.
- Patterns: point-to-point queues (SQS, RabbitMQ) for work distribution; publish-subscribe / event streaming (Kafka, SNS+SQS) for fan-out and replayability.
- Benefits: temporal decoupling (consumer can be down; messages queue), load leveling (absorb bursts), natural fit for event-driven workflows (an "ActivityUploaded" event triggers segment matching, feed generation, and analytics independently, with no single point of failure blocking upload acknowledgment).
- Costs: eventual consistency (the reader may see stale data briefly), harder end-to-end tracing/debugging, message ordering and idempotency become the caller's responsibility (duplicate delivery is normal — consumers must be idempotent), and operational overhead of running/monitoring brokers.
- Rule of thumb: **use sync for read-your-writes / user-facing request-response; use async for anything that "happens as a result of" an event and can tolerate seconds-to-minutes of lag.**

### Service Discovery — Client-Side vs Server-Side

Since service instances are ephemeral (autoscaling, rolling deploys, container rescheduling), callers cannot hardcode IP addresses. Service discovery maps a logical service name to live instance locations.

- **Client-side discovery**: the client queries a service registry directly (e.g., Netflix Eureka, Consul, Apache ZooKeeper) and applies load-balancing logic itself (e.g., round-robin among returned instances). Pro: no extra network hop, client controls LB algorithm. Con: registry-client library must be implemented/maintained per language; couples clients to the registry API.
- **Server-side discovery**: the client calls a fixed, well-known endpoint (a load balancer or reverse proxy — e.g., AWS ALB, Kubernetes Service + kube-proxy, an API gateway) which itself queries the registry and routes the request. Pro: clients stay simple/language-agnostic; registry logic centralized. Con: extra hop, LB becomes a scaling/HA-critical component.
- Modern default: **Kubernetes-native discovery** — a `Service` object gets a stable virtual IP/DNS name; kube-proxy (or a service mesh sidecar like Envoy in Istio/Linkerd) handles routing to healthy pod IPs. This is effectively server-side discovery with the "server" being the mesh data plane, and it additionally gives you mTLS, retries, and circuit breaking for free at the infrastructure layer rather than in application code.
- A **service registry** needs health checking (remove dead instances) and must tolerate registry unavailability gracefully (clients cache last-known-good instance lists).

### Data Ownership Boundaries — Why Shared Databases Are an Anti-Pattern

The **database-per-service** pattern says: each microservice owns its data exclusively, and every other service accesses that data only through the owning service's API (never via direct SQL/table access).

Why a shared database across services is an anti-pattern:
1. **Hidden coupling defeats independent deployability** — if Service A and Service B both read/write the same tables, a schema migration in A can silently break B; you no longer have independently deployable services, you have a distributed monolith with network latency added on top.
2. **No enforced invariants** — the owning service can no longer guarantee its own aggregate's business rules (e.g., "an effort's leaderboard rank is only updated through the ranking algorithm") because other services can write around it directly.
3. **Scaling and technology lock-in** — all services sharing one database must scale that database together and share its technology choice, even when access patterns differ wildly (e.g., leaderboard reads want a wide-column store; user profiles want relational integrity).
4. **Unclear ownership under incidents** — a shared table with five writers means an on-call engineer for any one service can't reason locally about what changed data or why.

Consequences you must design for once you commit to per-service data ownership:
- **No cross-service joins** — composing data from multiple services requires either API composition (caller queries each service and joins in memory/at the gateway) or a **CQRS read model** / materialized view that a consumer service builds asynchronously from other services' published events.
- **Distributed transactions** — you cannot use a single ACID transaction across services. Use the **Saga pattern** (a sequence of local transactions coordinated via choreography — each service publishes an event that triggers the next step — or orchestration, a central saga coordinator) with compensating actions for rollback.
- **Eventual consistency between services becomes a fact of life**, not a bug — reflected explicitly in your API contracts and UX (e.g., "your activity is processing" states).

## Case Study Solution: Strava-style Activity Platform

### Problem Statement & Clarifying Requirements

**Functional requirements:**
- Users upload a GPS-tracked activity (run/ride) from a mobile device (GPX/FIT file or live-tracked stream).
- The system processes the activity: computes distance/pace/elevation, matches GPS polyline segments against known "Segments" (popular routes), and updates segment leaderboards.
- Users can view leaderboards for a segment (all-time, this year, by age group, following-only).
- Users see a social feed of friends' activities and can give "kudos" / comment.
- Users have profiles (name, follower graph, stats).

**Non-functional requirements:**
- **Availability > strict consistency** for feed/kudos (social features tolerate a few seconds of staleness).
- **Durability** is critical for the raw uploaded activity file — never lose a user's recorded workout.
- **Low-latency upload acknowledgment** (user should see "upload received" within ~1-2s even though full processing takes longer).
- Segment matching must be **eventually consistent but correct** — leaderboards must converge to the right ranking even under out-of-order or retried updates (this mirrors the real consistency model Strava's engineering team designed for their leaderboard rebuild).
- System must handle highly non-uniform load: mass participation events (e.g., a Saturday morning group ride, or a marathon) spike uploads and segment-effort writes for a narrow set of segments.

### Capacity Estimation

Grounding on Strava's own disclosed numbers for a comparable system: ~1.4M activities/day, 75-100 new leaderboard "efforts" per second at peak, some individual segments with 500K+ recorded efforts and ~100K distinct athletes attempting them.

- Activities/day: 1.4M → ~16 activities/sec average; assume a 10x daytime peak factor → ~160/sec peak upload rate.
- Average GPX/FIT file size: ~200KB-1MB (1-2 hour activity at 1-5s GPS sampling). At 160/sec × 500KB avg ≈ 80 MB/s peak ingest bandwidth; ~1.4M × 500KB/day ≈ 700GB/day raw storage, ~250TB/year before considering multi-year retention and replication (3x replication ≈ 750TB/year).
- Segment-effort writes: 75-100/sec sustained, bursting far higher during mass events — this determines the write throughput requirement for the leaderboard store (favors a high-write-throughput wide-column store over a relational one).
- Feed reads dominate at read:write ratio easily 50:1 to 100:1 (users check feeds far more than they post) — favors heavy caching/CDN and read-optimized denormalized feed stores (fan-out-on-write for typical users, fan-out-on-read for celebrity/high-follower accounts, same as classic feed-scaling designs).
- Leaderboard queries are hot-and-skewed: a small number of popular segments (near cities) receive most read traffic — cache the top-N leaderboard per segment aggressively.

### High-Level Architecture

Bounded-context decomposition, each a separately deployable service with its own datastore:

```
                         ┌─────────────┐
    mobile/web  ───────► │ API Gateway  │
                         └──────┬───────┘
             ┌───────────────────┼────────────────────┬───────────────┐
             ▼                   ▼                    ▼               ▼
     ┌───────────────┐   ┌───────────────┐    ┌───────────────┐ ┌───────────────┐
     │  Activity      │   │  User Profile │    │ Segment /     │ │ Social Feed   │
     │  Ingestion Svc │   │  Service      │    │ Leaderboard   │ │ Service       │
     │  (sync upload) │   │  (sync CRUD)  │    │ Service       │ │               │
     └───────┬────────┘   └───────────────┘    └───────▲───────┘ └───────▲───────┘
             │ writes raw file to object store                  │               │
             │ publishes "ActivityUploaded"                     │               │
             ▼                                                  │               │
      ┌─────────────┐        Kafka / event bus                  │               │
      │ Object Store│◄───────────┬─────────────────────────────►┴───────────────┘
      │ (S3, blobs) │            │ async consumers subscribe to "ActivityUploaded"
      └─────────────┘            │ / "ActivityProcessed" / "SegmentEffortRecorded"
                                 │
                    ┌────────────┴────────────┐
                    │  GPS Processing Worker    │  (async: parse, compute stats,
                    │  (stats + segment match)  │   polyline-match against Segment
                    └────────────┬──────────────┘   index, emit SegmentEffortRecorded)
                                 │
                                 ▼
                    Segment/Leaderboard Service consumes,
                    writes ranked entry to Cassandra-like store
```

- **Activity Ingestion Service** (sync-facing): accepts the upload over REST, validates auth/quota, streams the raw file to object storage (S3), writes an `Activity` row (status=`processing`) to its own relational store, and publishes an `ActivityUploaded` event to Kafka. Responds 202 Accepted immediately — this is the key latency-hiding move.
- **GPS Processing Worker** (async, event-driven): consumes `ActivityUploaded`, parses the track, computes distance/elevation/pace, runs segment matching (geospatial index, e.g., R-tree/geohash lookup against known Segment polylines), and emits `ActivityProcessed` and one `SegmentEffortRecorded` event per matched segment.
- **Segment/Leaderboard Service**: owns Segment definitions and leaderboard entries; consumes `SegmentEffortRecorded` and applies an update (its own compute, following the real Strava design: re-derive current best effort from canonical storage rather than trusting message order, so out-of-order/duplicate delivery is safe). Exposes sync read APIs for leaderboard queries.
- **Social Feed Service**: consumes `ActivityProcessed` to fan out a feed entry to followers; owns kudos/comments; exposes sync read APIs for the feed and write APIs for kudos.
- **User Profile Service**: owns identity, follower graph, and stats; called synchronously by other services (e.g., Feed Service calls it to resolve follower lists) — a good candidate to expose as an **Open Host Service** with a stable API, since almost every other service depends on it (a Customer-Supplier relationship, Profile as supplier).

Sync vs async is deliberately mixed: upload acceptance, profile lookups, and leaderboard/feed reads are synchronous (user is waiting); everything that is a *consequence* of an upload (processing, matching, fan-out) is asynchronous via the event bus.

### API Design

```
POST /v1/activities
Headers: Authorization: Bearer <token>
Body: multipart/form-data { file: <gpx/fit>, device_ts, activity_type }
→ 202 Accepted
{ "activity_id": "act_9f2a", "status": "processing" }

GET /v1/activities/{activity_id}
→ 200 { "id": "act_9f2a", "status": "complete", "distance_m": 21195,
         "elevation_gain_m": 340, "matched_segments": [...] }

GET /v1/segments/{segment_id}/leaderboard?type=overall&gender=all&page=1
→ 200 {
    "segment_id": "seg_1044",
    "leaderboard_type": "overall",
    "entries": [
      { "rank": 1, "athlete_id": "u_501", "elapsed_time_s": 612, "effort_id": "eff_88" },
      { "rank": 2, "athlete_id": "u_233", "elapsed_time_s": 615, "effort_id": "eff_91" }
    ],
    "next_page_token": "opaque-cursor"
  }
```

`GET /activities/{id}` polling (or a WebSocket/push notification) lets the client move from "processing" to "complete" without blocking the initial upload response — this directly reflects the async processing boundary.

### Data Model (Per-Service — No Shared Database)

**Activity Ingestion Service (PostgreSQL)**
```
activities(id PK, athlete_id, status, raw_file_uri, activity_type,
           started_at, uploaded_at, distance_m, elevation_gain_m)
```
Owns only upload/processing status and computed summary stats; raw bytes live in object storage (S3), referenced by URI.

**Segment/Leaderboard Service (Cassandra-style wide-column store)**
```
segments(segment_id PK, name, polyline, city, sport_type)
leaderboard_entries(
  PARTITION KEY (segment_id, leaderboard_type),
  CLUSTERING KEY (elapsed_time_s, recorded_at, effort_id),
  athlete_id, effort_id, elapsed_time_s
)
-- secondary index on athlete_id for "my rank" lookups
```
This mirrors Strava's own published schema choice: partition by segment + leaderboard type so a leaderboard page is a single, pre-sorted partition scan; athlete-centric queries pay for a secondary index instead.

**Social Feed Service (denormalized document store, e.g., DynamoDB/Mongo)**
```
feed_entries(athlete_id PK, activity_id, author_id, summary_snapshot, created_at SK)
kudos(activity_id PK, athlete_id SK, created_at)
```
Feed is fanned out and denormalized (`summary_snapshot` is a copy, not a foreign key) precisely because Feed does not own Activity data — it consumes an `ActivityProcessed` event and stores what it needs, so it never queries Ingestion's database directly.

**User Profile Service (PostgreSQL)**
```
athletes(id PK, name, email, created_at)
follows(follower_id, followee_id, created_at, PRIMARY KEY(follower_id, followee_id))
```

No table in any of these schemas is written by more than one service. Cross-service data needs (e.g., Feed showing an athlete's name) are resolved either by a synchronous call to Profile Service or by including a denormalized snapshot at event-publish time — never by a foreign key into another service's schema.

### Deep Dive

**Justifying the bounded-context split:** The four services map to genuinely distinct subdomains with different consistency needs, data shapes, and scaling profiles. Activity Ingestion is I/O- and storage-heavy (large blobs, moderate write rate). Segment/Leaderboard is the **core domain** (Strava's actual competitive differentiator) — compute-heavy geospatial matching plus a very hot, skewed read pattern that justifies a specialized wide-column store, matching Strava engineering's own decision to move off Redis onto Cassandra for exactly this write-throughput and indexing reason. Social Feed is read-heavy and eventually-consistent by nature (nobody needs a kudos count to be linearizable). User Profile is comparatively low-volume, strongly-consistent, relational data (identity, follow graph) that many other services depend on as a stable supplier. Merging any two of these would either force an unnecessary technology compromise (e.g., leaderboard's wide-column needs vs. profile's relational integrity needs) or couple unrelated release cycles (feed-ranking experiments would gate leaderboard correctness fixes).

**Sync vs async justification:** Activity upload → 202 Accepted is synchronous only up to "durably stored + queued," never up to "fully processed" — because GPS parsing and segment matching (a geospatial computation against potentially thousands of nearby segment polylines) takes seconds to minutes and the user should not wait on the request thread. This is precisely the trigger described in the requirements: **activity upload triggers async segment matching.** Leaderboard and feed reads, by contrast, are synchronous REST calls because a user actively viewing a leaderboard needs a bounded-latency response now — but the *write side* that produces those rows is fed asynchronously from the event stream. Profile lookups (e.g., Feed Service resolving a follower list) are synchronous because the caller is already mid-request and the data is small/cacheable.

**Service discovery approach:** Given this system runs on Kubernetes-style infrastructure (the realistic modern default), server-side discovery via Kubernetes Services + a service mesh (Envoy/Istio or Linkerd) is the right fit: each service (Ingestion, Segment, Feed, Profile) gets a stable DNS name, the mesh's data-plane proxies handle load balancing across pod replicas, retries with backoff, circuit breaking (critical so a slow Segment Service doesn't cascade into Feed Service timeouts), and mTLS between services — all without embedding a discovery client library in each service's application code, which matters here because the services plausibly run different languages (a Go/Rust matching engine vs. a Rails/Django main app is common in this domain).

### Trade-offs and Alternatives Considered

- **Choreography vs orchestration for the upload→process→match→feed workflow**: chosen here as choreography (each service reacts to events independently) because the steps are largely independent fan-out, not a strict pipeline needing centralized rollback logic; a saga orchestrator would add complexity without a clear compensating-transaction need (a failed segment-match doesn't require "undoing" the upload).
- **Cassandra vs relational for leaderboards**: relational (Postgres) would be simpler operationally but cannot sustain the write-and-rank throughput at scale without heavy denormalization anyway — Strava's own migration away from a similarly limited approach (Redis) confirms a specialized store pays off at this scale; at smaller scale, a well-indexed Postgres table with materialized ranking is a reasonable starting point (monolith-first logic applies to store choice too).
- **Fan-out-on-write vs fan-out-on-read for feed**: fan-out-on-write (push to each follower's feed on publish) is used for typical accounts for read-time speed; high-follower accounts should switch to fan-out-on-read (compute at view time) to avoid a write storm — a classic hybrid social-feed trade-off.
- **Polling vs push for "activity processing" status**: a WebSocket/push notification gives a better UX than polling `GET /activities/{id}`, at the cost of maintaining stateful connections; many systems start with polling and add push later.

### How Real Systems Solve This

Strava's own engineering team published a detailed account of rebuilding their segment leaderboard infrastructure after their original Redis-based system (60 nodes, 1.8TB of memory) hit scaling and write-contention limits under aggressive locking; they moved to a Kafka-fed, Cassandra-backed design where Kafka messages are treated as *notifications* rather than authoritative state, and a consumer worker re-derives the correct leaderboard entry from canonical effort storage on each message — making the system tolerant of out-of-order and duplicate delivery, exactly the eventually-consistent pattern this design adopts. This is a strong real-world validation of "async event-driven update + re-derive-from-source-of-truth" over "trust message order," and of choosing a wide-column store for a write-heavy, read-skewed leaderboard workload.

## Sources

- [Rebuilding the Segment Leaderboards Infrastructure — Part 1: Background](https://medium.com/strava-engineering/rebuilding-the-segment-leaderboards-infrastructure-part-1-background-13d8850c2e77)
- [Rebuilding the Segment Leaderboards Infrastructure — Part 3: Design of the New System](https://medium.com/strava-engineering/rebuilding-the-segment-leaderboards-infrastructure-part-3-design-of-the-new-system-39fdcf0d5eb4)
- [Rebuilding the Segment Leaderboards Infrastructure — Part 4: Accessory Systems](https://medium.com/strava-engineering/rebuilding-the-segment-leaderboards-infrastructure-part-4-accessory-systems-5e98dc9a3d78)
- [Keeping Strava's Segment Leaderboards Fair: An Engineer's Perspective](https://stories.strava.com/articles/keeping-stravas-segment-leaderboards-fair-an-engineers-perspective)
- [Strava's GPS Data Processing Pipeline and Performance Testing](https://www.frugaltesting.com/blog/stravas-gps-data-processing-pipeline-and-performance-testing)
- [How to Architect a Fitness App with Social Features like Strava?](https://www.weblineindia.com/blog/build-fitness-app-like-strava/)
- [Design Strava: A Complete Guide](https://www.systemdesignhandbook.com/guides/design-strava/)
