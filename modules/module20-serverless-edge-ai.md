# Module 20: Serverless, Edge Computing & AI-Integrated Systems

## Core Concepts

### Lambda for event-driven workloads

AWS Lambda executes code in response to events (API Gateway requests, S3 puts, SQS messages, EventBridge rules, DynamoDB streams) without managing servers. Billing is per invocation and per GB-second of execution, with a max 15-minute timeout, making it a fit for bursty, short-lived, I/O-bound work rather than long-running or GPU-bound compute.

**Cold starts.** A cold start happens when Lambda must provision a new execution environment: download the deployment package/image, initialize the runtime, and run top-level (init) code, before the handler runs. Cold starts range from tens of milliseconds (compiled languages, small packages) to multiple seconds (JVM-based runtimes with large dependency graphs, or large container images). Warm invocations reuse a frozen execution environment and skip all of this. Concurrency scale-up, long idle periods, VPC-attached ENIs (historically a major tax, now mitigated by pre-warmed Hyperplane ENIs), and large deployment artifacts are the main cold-start amplifiers.

**Mitigation techniques, in order of strength:**
- **Provisioned Concurrency** — Lambda pre-initializes a specified number of execution environments and keeps them warm, so requests hit an already-initialized sandbox. It costs money whether invoked or not (billed like a reserved resource), and is typically paired with Application Auto Scaling to ramp provisioned capacity ahead of known traffic patterns (e.g., market open).
- **SnapStart** — available for Java (and now other runtimes) on Lambda: at deployment/publish time, Lambda initializes the function once, then takes an encrypted snapshot of the initialized execution environment's memory and disk state (via Firecracker microVM snapshotting). On subsequent cold starts, Lambda resumes from the snapshot rather than re-running init, cutting startup latency dramatically (often 90%+) at no extra steady-state cost, though a snapshot-resume can violate assumptions like unique random seeds or cached network connections, so re-initializing IDs/connections in an `afterRestore` hook is required.
- **Runtime choice** — Go, Rust, and Node.js/Python typically cold-start faster than JVM or .NET because interpreter/VM startup and class-loading overhead is smaller; keeping package size small, using Lambda layers judiciously, minimizing unnecessary imports, and avoiding VPC attachment unless required for private resource access all reduce cold-start latency further.
- **Keep-warm pings** are a weaker, legacy mitigation (a scheduled EventBridge rule invoking the function periodically) — it does not guarantee concurrency during a burst above the warm pool size, and AWS recommends Provisioned Concurrency instead for anything latency-sensitive.

### S3: presigned URLs, versioning, lifecycle policies

- **Presigned URLs** let a client (browser, mobile app) upload or download an S3 object directly, without proxying the bytes through your application and without exposing IAM credentials. The URL is signed (SigV4) with the issuer's credentials and an expiration (max 7 days for IAM-user/role-based presigning); anyone holding the URL before expiry can perform that single operation (GET/PUT) on that object. Common pattern: backend authorizes the request, generates a short-lived presigned PUT/GET, hands the URL to the client. Caveats: a clock-skew or a bucket-policy `Deny` can cause premature-looking expiration errors; presigned URLs inherit the permissions of the signer, so a compromised signer credential could presign broader access than intended if IAM scoping isn't tight.
- **Versioning** keeps every version of an object once enabled (cannot be fully disabled, only suspended), which protects against accidental overwrite/delete (a delete becomes a delete-marker, not a purge) and pairs with MFA Delete for compliance-grade protection of records like brokerage statements.
- **Lifecycle policies** automate the transition of objects across storage classes (Standard -> Standard-IA -> Glacier -> Glacier Deep Archive) and expiration of old versions/incomplete multipart uploads, driven by object age or tags — e.g., statements move to Glacier after 1 year and are retained 7 years for regulatory reasons, then expired.

### RDS: Multi-AZ failover, read replicas, automated backups

- **Multi-AZ** deployments synchronously replicate the primary to a standby in a different Availability Zone; on primary failure (AZ outage, instance failure, patching), RDS automatically fails over to the standby, typically completing in under a minute by flipping the DNS CNAME — the application does not need to change connection strings, only tolerate a brief connection drop/retry. Multi-AZ is for **availability**, not read scaling — the standby is not queryable in the single-standby model (the newer Multi-AZ DB Cluster variant does expose readable standbys).
- **Read replicas** use asynchronous (usually) replication to one or more replicas, which can be in the same region or cross-region, and are queryable — used to horizontally scale read traffic (e.g., portfolio history queries, reporting) off the primary. Replication lag means replicas are eventually consistent; reads that must reflect the latest write (e.g., "did my order just execute") must go to the primary.
- **Automated backups** take a daily snapshot plus continuous transaction-log shipping, enabling point-in-time restore to any second within the retention window (up to 35 days); this is distinct from manual/automated snapshots used for longer-term or cross-region DR copies.

### Edge computing: Cloudflare Workers vs. Lambda@Edge

Both let request-handling logic run at points of presence (PoPs) near the end user instead of a single origin region, cutting network round-trip time for latency-sensitive logic (auth checks, A/B routing, header rewriting, response caching, simple validation, geolocation-based redirects).

- **Cloudflare Workers** run on Cloudflare's own V8 isolate runtime deployed at (as of recent counts) 300+ cities globally; isolates start in single-digit milliseconds (no container/VM boot), support a `fetch`-style JS/Wasm API, and can attach to Durable Objects (strongly consistent, single-threaded actor storage), KV (eventually consistent global key-value), R2 (S3-compatible storage without egress fees), and D1 (edge SQLite). This makes Workers viable for more than routing — including lightweight stateful coordination.
- **AWS Lambda@Edge** runs standard Lambda functions (Node.js/Python) at CloudFront edge locations, hooked into four CloudFront trigger points (viewer request/response, origin request/response); it has tighter execution limits than regular Lambda (smaller memory/time ceilings, especially for viewer-request/response triggers) and pulls from the broader AWS ecosystem (IAM, VPC peering back to origin services) more naturally than Workers do. CloudFront Functions (a lighter, cheaper, sub-millisecond option) now handles simple viewer-request/response transformations that don't need Lambda@Edge's fuller runtime.
- **What belongs at the edge vs. origin:** stateless, cacheable, or purely request-shaping logic (auth token validation, geo-routing, header manipulation, static/cached quote snapshots, WAF-style filtering) is a good edge fit. Anything requiring strong consistency, multi-record transactions, or a system-of-record write (an authoritative financial ledger entry, an order execution decision) must not live at the edge — those require a single consistent origin datastore and cannot be safely resolved by whichever geographically nearest PoP a request happens to hit.

### AI-integrated system design: model serving and inference at scale

Serving ML/LLM models in production differs from typical CRUD services because compute is GPU-bound, often stateful (KV-cache for LLMs), and expensive to keep idle. Key patterns:
- **Batching** — dynamic/continuous batching (e.g., vLLM's continuous batching, NVIDIA Triton's dynamic batcher) groups concurrent inference requests so GPU compute is shared across them, dramatically increasing throughput per GPU versus one-request-at-a-time serving; the trade-off is a small added queuing latency per request.
- **Autoscaling GPU pools** — GPU autoscaling is slower and coarser-grained than CPU autoscaling (driver/model load time, GPU allocation scheduling), so systems typically keep a warm baseline pool sized to a p95 traffic floor and only autoscale the delta, or use a request queue with backpressure rather than scaling instantly to load.
- **Model serving patterns** — synchronous request/response for interactive use cases (chat, quote lookups), async/queue-based for expensive or non-latency-critical inference (nightly portfolio analysis), and a model router/gateway layer to select among model versions, apply rate limiting, and handle fallbacks.

### RAG architecture

Retrieval-Augmented Generation grounds an LLM's output in retrieved, private/current data rather than relying solely on parametric (trained-in) knowledge, reducing hallucination and enabling per-user personalization without retraining.

1. **Embedding pipeline** — source documents/records are chunked (by token count, semantic boundary, or structural markers), passed through an embedding model (e.g., OpenAI `text-embedding-3`, Cohere, or a self-hosted sentence-transformer) to produce dense vectors, and written to a vector store along with metadata (source id, ACL/owner, timestamp) for filtering.
2. **Vector store** — options span managed vector databases (Pinecone, Weaviate) offering approximate nearest neighbor (ANN) search (HNSW, IVF) as a service with built-in metadata filtering and horizontal scaling, versus **pgvector**, a Postgres extension that adds vector similarity search to an existing relational database — attractive when you want vectors co-located transactionally with the relational data they describe (e.g., a user's holdings), at the cost of scaling further than a single Postgres instance comfortably allows without extra sharding work.
3. **Retrieval** — at query time, the user's question (plus any conversation context) is embedded with the same model, an ANN search returns the top-k most similar chunks (often re-ranked by a cross-encoder for precision), filtered by the querying user's own ACL/tenant id so no cross-user data leaks.
4. **Prompt construction** — retrieved chunks are assembled into a context window alongside a system prompt (role, constraints, citation instructions) and the user question, respecting token budget by truncating/summarizing lowest-relevance chunks first.
5. **LLM call** — the composed prompt is sent to the LLM API (or self-hosted model); the response is optionally post-processed to attach citations back to source chunks and to reject/flag outputs that assert numeric facts not present in retrieved context (important where correctness matters).

## Case Study Solution: Robinhood-style Trading Platform

### Problem statement & requirements

Design a platform that streams real-time market quotes to users, executes buy/sell orders with strict correctness guarantees, stores account/statement documents, and offers an AI "portfolio insights" assistant that answers natural-language questions about a user's own holdings, history, and market context.

**Functional requirements**
- Stream live/near-live quotes for watchlisted and held symbols.
- Accept and execute market/limit orders with an auditable, idempotent order lifecycle (submitted -> routed -> filled/rejected).
- Generate/store account statements and tax documents (PDF) with secure, time-limited access.
- Portfolio assistant: answer "How has my portfolio performed this month?" or "What's my exposure to tech stocks?" using the user's real data plus market context, via RAG.

**Non-functional requirements**
- Quote delivery: <100ms end-to-end p95 from feed to client display.
- Order submission acknowledgment: <200ms p99; order execution routed to the exchange/broker-dealer layer with no correctness ambiguity (exactly-once semantics on the ledger).
- Strong consistency and durability for account balances, positions, and completed orders — no eventual consistency on money.
- Assistant responses: a few seconds is acceptable, but must never fabricate numbers about the user's own account.
- Auditability: every state transition on an order or balance is logged immutably (regulatory requirement, e.g., SEC/FINRA record-keeping).

### Capacity estimation

- Assume 5M DAU, of which 500K are actively watching live quotes at any given peak moment.
- Quote fan-out: 500K concurrent connections, each subscribed to ~10 symbols on average with market updates arriving every ~100-250ms per active symbol at market open → roughly 2-5M messages/sec at the edge/fan-out tier during the open, dropping to a small fraction of that at midday.
- Orders: peak of ~2,000 orders/sec at market open across the user base (vs. a few hundred/sec average through the day) — low absolute volume compared to quote traffic, but each one requires strict correctness, so it is provisioned and tested for its own peak rather than an average.
- AI assistant: assume 2% of DAU issue ~1 assistant query/day → ~100K queries/day, ~1-2 QPS average with bursts around market close/tax season; each query does 1 embedding call + 1 vector search (top-k ~8-15) + 1 LLM call (a few hundred ms to a few seconds).
- Storage: statements/documents at ~5MB/user/year × 5M users ≈ 25TB/year in S3, lifecycle-transitioned to Glacier after a year.

### High-level architecture

```
                       ┌─────────────────────────┐
 Market Data Feed ───► │  Streaming Ingest        │
 (exchange/vendor)     │  (Kinesis/Kafka)         │
                       └──────────┬───────────────┘
                                  │
                    ┌─────────────▼──────────────┐
                    │  Quote Fan-out Service       │
                    │  (WebSocket gateway, region) │
                    └─────────────┬────────────────┘
                                  │ push
                    ┌─────────────▼──────────────┐
                    │   Edge Layer (CDN PoPs)      │
                    │  Cloudflare Workers /        │
                    │  Lambda@Edge: connection     │
                    │  routing, auth-token check,  │
                    │  geo-nearest gateway pick     │
                    └─────────────┬────────────────┘
                                  │
                             Client (app/web)

 Client ── place order ──► API Gateway ──► Lambda (order intake,
                                            idempotency check,
                                            provisioned concurrency)
                                              │
                                        SQS/EventBridge (event-driven)
                                              │
                                   Order Matching/Routing Service
                                   (long-running service, NOT edge,
                                    NOT bare Lambda for the hot path—
                                    or Lambda w/ SnapStart+provisioned
                                    concurrency if kept serverless)
                                              │
                                   ┌──────────▼───────────┐
                                   │ RDS Multi-AZ (Postgres)│
                                   │ accounts/orders/holdings│
                                   │ + read replicas for     │
                                   │   history/reporting     │
                                   └────────────────────────┘

 Statements/Docs ──► S3 (versioned, lifecycle to Glacier)
                      served via presigned GET URLs

 AI Assistant:
 Client ── question ──► API Gateway ─► Lambda (RAG orchestrator)
      │
      ├─► Embedding API (query embedding)
      ├─► Vector Store (pgvector or Pinecone) — top-k retrieval
      │      of user's holdings/transactions snapshots + doc chunks
      ├─► Read replica (fresh structured facts: balances, positions)
      └─► LLM API — prompt = system + retrieved context + question
                → response with numeric facts sourced from step above,
                  not from the LLM's own generation
```

### API design

```
GET  /v1/quotes/{symbol}                 -> latest quote (REST fallback)
WS   /v1/quotes/stream?symbols=AAPL,TSLA -> streaming quote subscription

POST /v1/orders
  { "symbol": "AAPL", "side": "buy", "type": "limit",
    "qty": 10, "limit_price": 190.25, "idempotency_key": "uuid" }
  -> 202 Accepted { "order_id", "status": "submitted" }

GET  /v1/orders/{order_id}               -> order status/lifecycle

GET  /v1/documents/{doc_id}/download-url -> { "url": presigned S3 GET, "expires_in": 300 }

POST /v1/assistant/query
  { "question": "How concentrated am I in tech stocks?" }
  -> { "answer": "...", "sources": [...], "as_of": "2026-09-05T14:00:00Z" }
```

The order endpoint requires an `idempotency_key` so retried client requests (e.g., after a timeout) never double-submit an order — the intake Lambda checks/stores the key transactionally before enqueuing.

### Data model

```sql
accounts(id PK, user_id, cash_balance_cents, buying_power_cents, status, updated_at)

orders(id PK, account_id FK, symbol, side, type, qty, limit_price,
       status,            -- submitted|routed|partially_filled|filled|rejected|canceled
       idempotency_key UNIQUE,
       created_at, updated_at)

fills(id PK, order_id FK, qty, price, executed_at)

holdings(account_id FK, symbol, qty, avg_cost_cents,
         PRIMARY KEY(account_id, symbol))

documents(id PK, account_id FK, s3_key, doc_type, version, created_at)
```

```sql
-- RAG vector store (pgvector example)
portfolio_embeddings(
  id PK,
  account_id,               -- for row-level ACL filtering
  source_type,              -- 'holding_snapshot' | 'statement_chunk' | 'transaction'
  source_id,
  chunk_text,
  embedding VECTOR(1536),
  as_of_date,
  created_at
)
-- index: CREATE INDEX ON portfolio_embeddings USING hnsw (embedding vector_cosine_ops);
-- every query filters WHERE account_id = :current_user_account_id first,
-- then does the ANN search — never rely on the vector search alone for isolation.
```

### Deep dive

**Why cold starts are dangerous here, and mitigation.** A cold start on the order-intake path directly adds latency to a user's buy/sell decision at exactly the moment (market open, a fast-moving stock) when price can move against them within that added window — a few hundred extra milliseconds of cold-start latency is a real-money slippage risk, not just a UX annoyance. Mitigation: keep the order-intake Lambda's package minimal and dependency-light, choose a fast-starting runtime, and hold **Provisioned Concurrency** sized to the historical p99 concurrent-order volume at market open (scheduled to scale up automatically via Application Auto Scaling ahead of 9:30am ET), with **SnapStart** as an additional layer if running on a JVM-based runtime for this path. The actual order-matching/routing engine, however, is typically not bare Lambda at all in a real trading system — it is a long-running, warm, horizontally-scaled service (containers/VMs) precisely because "no cold start, ever" is a hard requirement for the authoritative execution path; Lambda is better suited to the surrounding event-driven work (validation, notification, statement generation, fraud checks) than to the matching engine itself.

**Where edge computing helps, and where it can't.** The edge layer is well-suited to quote delivery: routing a user's WebSocket connection to the geographically nearest fan-out gateway, validating auth tokens before establishing a stream, and serving very-short-TTL cached snapshot quotes for less time-sensitive UI (e.g., a watchlist screen that refreshes every few seconds) — all of this cuts the network leg of latency without touching the source of truth. It cannot be used for the authoritative act of executing or recording an order: an edge PoP has no consistent view of the user's real-time buying power or the exchange's order book, and running execution logic redundantly at dozens of PoPs would create race conditions and reconciliation nightmares. Order submission may originate at the edge (initial request routing, auth), but the decision of whether the order is valid and how it fills must resolve at a single authoritative origin service backed by the RDS Multi-AZ primary.

**RAG pipeline walkthrough for the portfolio assistant.** (1) The user's question is embedded with the same embedding model used to index portfolio data. (2) A vector similarity search over `portfolio_embeddings`, always pre-filtered by `account_id`, retrieves the top-k most relevant chunks — recent transaction summaries, current holdings snapshots, relevant statement excerpts. (3) In parallel, the orchestrator fetches authoritative structured numbers (current balance, positions) directly from an RDS read replica rather than trusting embedded/stale numeric text, because financial figures must be exact, not approximately retrieved. (4) A prompt is constructed: a system message instructing the model to answer only from the provided context and to never invent figures, the retrieved chunks, the fresh structured facts, and the user's question. (5) The LLM API call returns a natural-language answer; the response is validated by cross-checking any numeric claims against the structured facts pulled in step 3, and citations to source documents are attached before returning to the client.

### Trade-offs and alternatives considered

- **Serverless order intake vs. dedicated service for matching**: serverless everywhere is simpler operationally but cold starts and cost-at-sustained-load make a dedicated always-on service the right choice for the matching engine, while serverless remains appropriate for the bursty, tolerant-of-a-few-hundred-ms edges of the system.
- **pgvector vs. managed vector DB (Pinecone/Weaviate)**: pgvector keeps portfolio vectors transactionally co-located with relational account data and avoids a second system to operate, but scales less gracefully past tens of millions of vectors; a managed vector DB scales further and offers richer ANN tuning, at the cost of an extra network hop and a second system to keep in sync/ACL-consistent with the source of truth.
- **Lambda@Edge vs. Cloudflare Workers for the quote-routing edge layer**: if the platform is already AWS-native (CloudFront, IAM, VPC), Lambda@Edge integrates more naturally; if minimizing edge cold-start (isolate startup is near-instant) and lowering egress cost matter more, Cloudflare Workers is the stronger fit — many real systems end up using CloudFront Functions or a thin Workers layer purely for routing/auth, keeping heavier logic at origin regardless of which edge platform is chosen.
- **Multi-AZ alone vs. adding cross-region DR**: Multi-AZ protects against AZ failure with automatic failover but not regional failure; a regulated trading platform typically layers cross-region read replicas or backup restoration procedures on top, accepting the added replication cost for disaster-recovery posture.

### How real systems solve this

Public system-design write-ups of Robinhood-style platforms describe a similar shape: a low-latency streaming layer for market data fan-out, a strongly consistent ledger/order service decoupled from the noisier read-heavy paths (watchlists, quote history) via CQRS-like separation, and increasingly a RAG-based assistant layered on top of the existing data lake/warehouse (Robinhood's own engineering posts describe a large-scale data lake as the backbone for analytics and product features) rather than a wholly separate pipeline — i.e., production RAG features are typically built to read from the same governed data platform the rest of the product already trusts, rather than a parallel ad hoc export, precisely so the "don't fabricate the user's own numbers" requirement is met by construction.

## Sources

- [Improving startup performance with Lambda SnapStart — AWS Lambda docs](https://docs.aws.amazon.com/lambda/latest/dg/snapstart.html)
- [AWS Lambda Cold Start Mitigation Guide — Provisioned Concurrency, SnapStart, and Code-Level Techniques](https://hidekazu-konishi.com/entry/aws_lambda_cold_start_mitigation_guide.html)
- [Download and upload objects with presigned URLs — Amazon S3 User Guide](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html)
- [Sharing objects with presigned URLs — AWS Documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html)
- [Resilience in Amazon RDS — Amazon RDS User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/disaster-recovery-resiliency.html)
- [Amazon RDS Read Replicas — AWS](https://aws.amazon.com/rds/features/read-replicas/)
- [AWS — Difference between Multi-AZ and Read Replicas in Amazon RDS](https://medium.com/awesome-cloud/aws-difference-between-multi-az-and-read-replicas-in-amazon-rds-60fe848ef53a)
- [How Can Serverless Computing Improve Performance? — Cloudflare Learning](https://www.cloudflare.com/learning/serverless/serverless-performance/)
- [Edge Computing: AWS Lambda@Edge vs. Cloudflare Workers — A Practical Guide](https://mkabumattar.com/blog/post/edge-computing-aws-lambda-at-edge-vs-cloudflare-workers-practical-guide/)
- [Data Lake at Robinhood — Robinhood Newsroom](https://robinhood.com/us/en/newsroom/data-lake-at-robinhood/)
- [Scaling and Governing Robinhood's Data Lakehouse — Onehouse](https://www.onehouse.ai/blog/scaling-and-governing-robinhoods-data-lakehouse)
- [Robinhood System Design Interview: The Complete Guide](https://www.systemdesignhandbook.com/guides/robinhood-system-design-interview/)
- [Building Reliable RAG Pipelines with Pinecone](https://medium.com/@ankurnitp/practical-guide-how-rag-works-with-pinecone-f792801c946e)
- [Should You Use pgvector or Pinecone for RAG in 2026?](https://buildspace.site/blog/postgresql-pgvector-vs-pinecone-rag-2026)
- [We Tried and Tested 10 Best Vector Databases for RAG Pipelines — ZenML Blog](https://www.zenml.io/blog/vector-databases-for-rag)
