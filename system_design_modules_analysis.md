# System Design Modules — Complete Analysis

**Directory:** `c:\Users\Bindukumar\Downloads\system-design-modules-individual`
**Total Modules:** 24 | **Total Size:** ~620 KB

---

## Overall Structure & Pedagogical Pattern

Every module follows an identical 3-part structure:

| Section                 | Purpose                                                               |
| ----------------------- | --------------------------------------------------------------------- |
| **Core Concepts**       | Theory — definitions, trade-offs, patterns, math                      |
| **Case Study Solution** | Requirements → Capacity → Architecture → API → Data Model → Deep Dive |
| **Sources**             | Primary references (engineering blogs, RFCs, AWS/Google docs)         |

The case studies apply theory to recognizable real-world products using the **PEDALS / RESHADED** interview frameworks.

---

## Module-by-Module Summary

### Module 01 — System Design Thinking

**Theory:** PEDALS framework, back-of-envelope math, CAP theorem, consistency models.
**Case Study:** WhatsApp — stateless REST for accounts + stateful WebSocket for messaging.

### Module 02 — Scalability Fundamentals

**Theory:** Horizontal vs. vertical scaling, sharding (range/hash/directory), consistent hashing, LB algorithms.
**Case Study:** Twitter/X — fan-out-on-write vs. fan-out-on-read; celebrity hybrid pattern; Redis sorted sets.

### Module 03 — Database Fundamentals

**Theory:** ACID vs. BASE, B-Tree vs. LSM-Tree, SQL vs. NoSQL selection, JSONB.
**Case Study:** Airbnb — relational schema for bookings, SQL for transactional correctness.

### Module 04 — Database Scaling

**Theory:** Read replicas, sharding, partitioning, PgBouncer, DynamoDB GSI design.
**Case Study:** Uber Eats — order lifecycle, sharding by `restaurant_id`, hot-partition mitigation.

### Module 05 — Caching Fundamentals

**Theory:** Cache-aside/read-through/write-through/write-back; eviction (LRU/LFU/ARC); TTL; stampede mitigation.
**Case Study:** Reddit — hot-post caching, probabilistic early expiration.

### Module 06 — Distributed Caching & CDN

**Theory:** Redis Cluster (hash slots), consistent hashing, CDN edge caching, cache invalidation.
**Case Study:** Netflix — Open Connect Appliances, proactive cache pre-positioning, origin shield.

### Module 07 — Messaging Systems

**Theory:** AMQP vs. Kafka, delivery guarantees (at-most/at-least/exactly-once), Kafka internals, DLQ, backpressure.
**Case Study:** Tinder — Kafka for swipe events, per-user partitioning, DLQ for failed notifications.

### Module 08 — Event-Driven Architecture & CQRS

**Theory:** Event Sourcing, CQRS, Saga (choreography vs. orchestration), Outbox pattern, Debezium CDC.
**Case Study:** Online Judge — event-sourced submission lifecycle, CQRS for leaderboard, Saga for test runners.

### Module 09 — API Architectures

**Theory:** REST, GraphQL (N+1 / DataLoader), gRPC (Protobuf/streaming), API versioning.
**Case Study:** WhatsApp API — REST externally, gRPC internally, WebSocket for real-time delivery.

### Module 10 — Real-Time & Async APIs

**Theory:** WebSockets (bidirectional), SSE (server→client only), Long Polling, Webhooks; degradation strategies.
**Case Study:** Yelp real-time reviews — SSE for review streaming, WebSocket for owner chat.

### Module 11 — Microservices Fundamentals

**Theory:** DDD bounded contexts, service discovery, service-per-database, API gateway, sync vs. async comms.
**Case Study:** Strava — Activity/Social/Analytics bounded contexts, event bus for social feed fan-out.

### Module 12 — Advanced Microservices

**Theory:** Circuit breaker (resilience4j), bulkhead, retry with backoff+jitter, Strangler Fig, Sidecar/Istio, rate limiters.
**Case Study:** Rate Limiter — Redis token bucket, sliding-window-log for precision, leaky bucket for smoothing.

### Module 13 — Fault Tolerance & Resilience

**Theory:** Active-active/passive HA, "nines" table, bulkhead, fail-fast vs. retry, graceful degradation, RTO/RPO, Chaos Monkey/Kong.
**Case Study:** Online Auction — single-writer-per-auction linearizability, idempotency keys, RPO=0 bid ledger, notification service gracefully degraded.

### Module 14 — Observability & Monitoring

**Theory:** Metrics/Logs/Traces pillars; OTel + W3C traceparent; RED/USE methods; SLI/SLO/SLA; error budgets; blameless postmortems; FinOps.
**Case Study:** Facebook Live Comments — tail-based sampling, error-budget gating deploys during live events, correlation-ID root-cause analysis.

### Module 15 — Security Architecture

**Theory:** Defense in depth (7 layers), Zero Trust, OAuth2+OIDC+PKCE, JWT/JWKS at edge, Vault dynamic secrets, mTLS, OWASP API Top 10 (2023).
**Case Study:** Facebook Post Search — live ACL evaluation at query time (not index time) prevents BOLA (API1); Unicorn-style graph intersection for FRIENDS.

### Module 16 — Compliance & Protection

**Theory:** WAF (positive/negative model), RBAC vs. ABAC, OWASP Top 10 → architectural responses, GDPR/SOC2/HIPAA, SBOM (Syft/Grype/Snyk), SLSA provenance.
**Case Study:** Price Tracker — RBAC+ABAC hybrid, JA4 bot fingerprinting, EU data residency, SBOM for scraping-library dependencies.

### Module 17 — Cloud Architecture (AWS)

**Theory:** EC2 vs. ECS/Fargate vs. Lambda selection, S3/RDS/Aurora/ElastiCache, VPC (public/private/NAT), ALB vs. NLB, SQS/SNS patterns, Terraform, cost optimization.
**Case Study:** Instagram on AWS — presigned S3 PUT (never proxy video through API), S3→SNS→SQS fan-out, three-tier VPC layout.

### Module 18 — Containers & Kubernetes

**Theory:** Docker multi-stage builds, K8s architecture (control plane/kubelet/kube-proxy), HPA+KEDA, Helm, GitOps (ArgoCD/Flux), rolling updates, Trivy image scanning.
**Case Study:** YouTube Top-K Trending — Count-Min Sketch + Space-Saving, StatefulSet for partition-affinity, KEDA on Kafka consumer lag.

### Module 19 — CI/CD & Platform Engineering

**Theory:** 4-stage pipeline (commit→build→deploy→progressive delivery), rolling/blue-green/canary trade-offs, feature flags (LaunchDarkly/AppConfig), Backstage IDP.
**Case Study:** Uber Deployments — μDeploy auto-rollback on metrics, per-city feature flags for dispatch, Backstage for 4,000 microservices.

### Module 20 — Serverless, Edge & AI

**Theory:** Lambda cold starts (Provisioned Concurrency, SnapStart), S3 presigned URLs, RDS Multi-AZ vs. read replicas, Cloudflare Workers vs. Lambda@Edge, RAG pipeline, pgvector.
**Case Study:** Robinhood — Provisioned Concurrency for market-open order intake, edge for WebSocket routing (not execution), RAG portfolio assistant with ACL vector filtering.

### Module 21 — URL Shortener Build

**Build:** Snowflake IDs → Base62, 302 (not 301) for analytics, partial Postgres index, Redis allkeys-lru cache-aside, SQS async click events.

### Module 22 — URL Shortener Productionizing

**Build:** ALB least-outstanding-requests, separate target groups (create vs. redirect), target-tracking ASG, CDN 301/302 caching gotcha, token-bucket rate limiting, canary CI/CD.

### Module 23 — Real-Time Chat Build

**Build:** 5M WebSockets / ~100 gateways, Kafka (durable fan-out) + Redis Pub/Sub (cross-instance socket routing), TTL-based presence, watermark read receipts, cursor pagination.

### Module 24 — Scaling Chat Production

**Build:** GeoDNS + per-region Kafka + MirrorMaker2 cross-region relay, globally-replicated TTL presence, presigned POST S3 uploads, notification pipeline reusing Kafka backbone, chaos testing checklist.

---

## Curriculum Architecture (5 Tracks)

```
FOUNDATION (01–06)
  Thinking → Scalability → DB Fundamentals → DB Scaling → Caching → Distributed Cache

DISTRIBUTED SYSTEMS PRIMITIVES (07–12)
  Messaging → Event-Driven/CQRS → API Patterns → Real-Time → Microservices → Advanced μServices

OPERATIONAL EXCELLENCE (13–16)
  Fault Tolerance → Observability → Security → Compliance

CLOUD & PLATFORM (17–20)
  AWS Cloud → Kubernetes → CI/CD → Serverless+AI

FULL-STACK BUILDS (21–24)
  URL Shortener Build → URL Shortener Production → Chat Build → Chat Production
```

---

## Cross-Cutting Themes

| Theme                                                   | Modules                        |
| ------------------------------------------------------- | ------------------------------ |
| Idempotency keys                                        | 07, 08, 13, 20, 21, 23         |
| Kafka partitioning strategy                             | 07, 08, 14, 18, 23, 24         |
| Redis patterns (cache-aside, pub/sub, sorted sets, TTL) | 05, 06, 12, 17, 21, 22, 23, 24 |
| Presigned S3 URLs                                       | 17, 20, 24                     |
| Snowflake IDs / ULID                                    | 08, 21, 23                     |
| Cursor-based pagination                                 | 03, 15, 23                     |
| Fan-out-on-write vs. read                               | 02, 14, 17, 23                 |
| Circuit breaker + bulkhead                              | 12, 13                         |
| mTLS + Zero Trust                                       | 15, 16                         |
| Canary deployments                                      | 19, 22                         |
| Chaos engineering                                       | 13, 24                         |
| SLI/SLO/Error budgets                                   | 13, 14                         |
| OWASP API Security                                      | 15, 16                         |

---

## Case Study Reference Table

| Module | Case Study               | Problem Solved                                            |
| ------ | ------------------------ | --------------------------------------------------------- |
| 01     | WhatsApp                 | Interview framework (PEDALS)                              |
| 02     | Twitter/X                | Feed fan-out at scale                                     |
| 03     | Airbnb                   | Transactional booking correctness                         |
| 04     | Uber Eats                | Hot-partition DB sharding                                 |
| 05     | Reddit                   | Cache stampede prevention                                 |
| 06     | Netflix                  | Multi-tier CDN proactive caching                          |
| 07     | Tinder                   | Ordered swipe-event processing                            |
| 08     | Online Judge             | Event-sourced submission lifecycle                        |
| 09     | WhatsApp API             | API protocol selection                                    |
| 10     | Yelp Real-time           | SSE vs WebSocket vs polling                               |
| 11     | Strava                   | Bounded context decomposition                             |
| 12     | Rate Limiter             | Token bucket / sliding window                             |
| 13     | Online Auction           | Linearizable bids + bulkhead                              |
| 14     | FB Live Comments         | Observability-driven regression detection                 |
| 15     | FB Post Search           | Privacy-aware search (BOLA prevention)                    |
| 16     | Price Tracker            | WAF + ABAC + supply-chain security                        |
| 17     | Instagram on AWS         | VPC layout + S3 async upload                              |
| 18     | YouTube Top-K            | Count-Min Sketch + StatefulSet K8s                        |
| 19     | Uber Deployments         | Canary + feature flags + IDP                              |
| 20     | Robinhood                | Serverless + RAG + edge computing                         |
| 21     | URL Shortener            | Snowflake IDs + Redis cache-aside                         |
| 22     | URL Shortener Prod       | Auto-scaling + CDN 301/302 gotcha                         |
| 23     | Chat System              | WebSocket + Kafka + Redis Pub/Sub                         |
| 24     | Chat Production          | Multi-region + presence + S3 uploads                      |
| 25     | Google Drive / Dropbox   | Delta sync, chunking, block vs metadata storage           |
| 26     | Uber / Lyft Ride-Sharing | Geospatial indexing (QuadTrees / Geohashing)              |
| 27     | Ticketmaster Booking     | High-concurrency virtual waiting rooms, Redis Redlock     |
| 28     | Web Crawler              | BFS traversal, URL frontier, Bloom filters for duplicates |
| 29     | Zoom / Video Streaming   | WebRTC, UDP over TCP, SFU networking                      |

---

## Key "Aha" Insights Per Track

### Databases

- **Partial indexes** (`WHERE is_active = TRUE`) keep hot-path indexes small
- **Watermark read receipts** = O(members) writes vs. O(messages × members) for per-message receipts
- **LSM trees** beat B-Trees for high-write-throughput append workloads (Cassandra/RocksDB)

### Caching

- **Cache hit ratio** is a _leading indicator_ of DB load — monitor it before CPU spikes
- Redis **Pub/Sub loses messages if no subscriber** — never the sole delivery mechanism
- `allkeys-lru` eviction for URL shortener; TTL-based expiry is a natural recency signal

### Messaging

- **Kafka + Redis Pub/Sub are complementary**: Kafka = durable + replayable; Redis Pub/Sub = ephemeral + real-time cross-process routing
- Partition by `conversation_id` for ordering; by `user_id` for per-user parallelism

### APIs & Real-Time

- **302 over 301** for mutable links — 301 cached in browsers indefinitely with no client-side invalidation
- **SSE is server→client only**; WebSockets are bidirectional — SSE cannot handle typing indicators
- **Presigned POST** enforces server-side file size/type limits without proxying bytes through your API

### Resilience

- **RTO ≠ RPO**: fast failover (low RTO) to slightly stale data (high RPO) is a valid design choice
- **Single-writer-per-partition** achieves linearizability without distributed consensus overhead
- **Chaos engineering** turns resilience claims into continuously-verified facts rather than assumptions

### Security

- **Dynamic secrets** (Vault) > short-lived rotated secrets > static credentials
- **BOLA (OWASP API1)** requires re-evaluating authorization _per object on the response path_, not just at the route level
- **mTLS** gives encryption-in-transit AND workload identity simultaneously

### CI/CD

- **Feature flags decouple deploy (infra risk) from release (business risk)** — orthogonal levers
- **In-process flag evaluation** is mandatory at high eval rates (no network call per flag check)
- **Canary bounds blast radius mathematically** — 1% traffic = max 1% of users affected during bake

---

## Interview Readiness Summary

These 24 modules cover the full scope of senior/staff-level system design interviews:

- **Classic designs:** URL shortener, chat, social feed, search, rate limiter, live comments
- **Probabilistic data structures:** Count-Min Sketch, Bloom filters, Snowflake IDs, sorted sets
- **AWS depth:** EC2/ECS/Lambda selection, VPC layout, ALB/NLB, SQS/SNS, Aurora, ElastiCache
- **Kubernetes depth:** StatefulSets, HPA/KEDA, GitOps, Helm, rolling updates, image scanning
- **Security depth:** PKCE, mTLS, ABAC, OWASP API Top 10, SBOM/supply chain, Zero Trust
- **Observability:** OTel W3C traceparent, Prometheus PromQL, SLO/error-budget policy
- **Trade-off fluency:** Every case study has an explicit "Trade-offs and Alternatives Considered" section

---

## The Absolute Best Ways to Fast-Track System Design Interview Prep

If you need to optimize your time to get interview-ready as quickly as possible, here is the proven fast-track methodology:

### 1. Memorize ONE Framework (and stick to it)

An interviewer is grading you on your structured approach as much as your technical knowledge. Memorize a framework like **PEDALS**:

- **P**roblem Requirements (Functional & Non-functional)
- **E**stimation (Back-of-the-envelope math)
- **D**esign (High-level architecture drawing)
- **A**PIs (Define the contract: REST/gRPC/WebSocket)
- **L**atency & Availability (Scaling out, Caching, Load Balancing)
- **S**cale (Database sharding, partitioning)

### 2. Master the "Big 4" Building Blocks

Instead of trying to learn every tool, master the trade-offs of the foundational four:

- **Load Balancing & API Gateway:** Nginx, ALB/NLB. Know when to use Round Robin vs. IP Hash.
- **Caching:** Redis/Memcached. Know Cache-Aside vs. Write-Through, and eviction policies (LRU).
- **Databases:** Know when to use Relational (PostgreSQL) vs. NoSQL (Cassandra/DynamoDB).
- **Message Queues:** Kafka vs. RabbitMQ/SQS. Know that Kafka is for durable, replayable event streaming.

### 3. Learn the 5 Archetypal Systems

Almost every interview question is a variation of these five patterns:

1. **Read-Heavy / Fan-Out:** (e.g., Twitter Feed). _Key concept: Caching, Pre-computing feeds, CDN._
2. **Write-Heavy / Append-Only:** (e.g., Metrics/Logging). _Key concept: Cassandra/LSM Trees, Kafka._
3. **Real-Time / Bidirectional:** (e.g., WhatsApp, Discord). _Key concept: WebSockets, Presence Servers._
4. **Proximity / Geospatial:** (e.g., Uber, Yelp). _Key concept: QuadTrees, Geohashing._
5. **Stateful / Transactional:** (e.g., Ticketmaster). _Key concept: ACID, Distributed Locks, Idempotency._

### 4. Practice Out Loud (Mock Interviews)

System design is a conversation, not a coding test. You must practice drawing while explaining your trade-offs out loud.

### 5. Review "Cheat Sheets"

Review the 29 case studies generated in the `case_studies/` directory. They contain the specific "aha!" insights that elevate a candidate from a "Hire" to a "Strong Hire".

> **Total content:** ~620 KB across 29 case study files, all backed by primary sources — engineering blogs from Netflix, Uber, Slack, Discord, Meta, Twitter, and Google SRE.
