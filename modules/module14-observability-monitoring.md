# Module 14: Observability & Monitoring

## Core Concepts

### The Three Pillars of Observability

**Metrics** are numeric, time-series aggregates (counters, gauges, histograms) recorded at fixed intervals. They are cheap to store and query, compress well, and are the right tool for dashboards, alerting thresholds, and long-term trend analysis (e.g., "p99 latency over the last 30 days"). Their weakness: they are pre-aggregated, so you lose per-request detail — a metric can tell you *that* error rate spiked at 14:03 but not *which* request or *why*. Prometheus's four core types (Counter, Gauge, Histogram, Summary) are the de facto standard.

**Logs** are discrete, timestamped, often unstructured or semi-structured events emitted by application code. They carry the richest per-event detail (stack traces, request payloads, decision branches) and are indispensable for root-cause forensics after an alert fires. Their weakness: high cardinality and volume make them expensive to store and slow to query at scale, and unstructured logs (plain text) are hard to aggregate or alert on reliably.

**Traces** capture the causal, end-to-end path of a single request as it moves through multiple services, broken into a tree of **spans**, each with a start time, duration, and metadata (tags/attributes). Traces are the only pillar that shows *where time went* across a distributed call graph and expose which downstream dependency caused a slow or failed request. Their weakness: sampling is usually required at high volume (100% tracing at millions of req/s is cost-prohibitive), so a specific failing request may not have been captured unless tail-based sampling is used.

In practice the three are complementary and correlated via shared identifiers: a metric spike triggers an alert → the alert links to a Grafana dashboard and to exemplar trace IDs → a trace shows the slow span → the trace's correlation ID is used to pull matching structured logs for the exact request. This "correlated telemetry" workflow is the practical definition of observability, as opposed to traditional monitoring which only watches predefined dashboards for known failure modes.

### Structured Logging with Correlation IDs

Structured logging emits logs as machine-parseable key-value records (JSON, logfmt) rather than free text, e.g. `{"ts":..., "level":"error", "service":"comment-api", "trace_id":"...", "user_id":..., "msg":"write timeout"}`. This makes logs queryable (via Loki, Elasticsearch, CloudWatch Insights) the same way a database table is queryable, instead of relying on regex/grep.

A **correlation ID** (often the same as the OpenTelemetry `trace_id`) is generated at the edge (API gateway or load balancer) on the first hop of a request and propagated through every downstream call — via HTTP headers, gRPC metadata, or message headers on a queue. Every log line, span, and metric label emitted while handling that request includes this ID. This is what lets an engineer take one failed request reported by a user, find its trace, and pull every log line across every microservice that touched it, in causal order. Without a correlation ID, cross-service debugging degrades into manually correlating logs by approximate timestamp — unreliable under concurrency and clock skew.

### Distributed Tracing via OpenTelemetry

OpenTelemetry (OTel) is the CNCF-backed, vendor-neutral standard for instrumentation, unifying what used to be separate metrics/tracing/logging SDKs (subsuming OpenTracing and OpenCensus). Key concepts:

- **Span**: the fundamental unit of tracing — a named, timed operation with a unique `span_id`, a `trace_id` shared by the whole request, a parent `span_id` (forming a tree), and attributes/events/status. A span for "handle POST /comments" might have child spans for "write to Kafka" and "call moderation service."
- **Trace**: the full tree of spans for one logical request/transaction, identified by a single `trace_id`.
- **Context propagation**: the mechanism that carries the active trace context (trace ID, span ID, sampling decision) across process/network boundaries so child spans attach to the right parent. OpenTelemetry implements this via the **W3C Trace Context** standard, primarily the `traceparent` HTTP header: `traceparent: 00-{trace-id}-{parent-span-id}-{trace-flags}`, plus an optional `tracestate` header for vendor-specific data. This standardization is what lets Prometheus, Jaeger, Datadog, Honeycomb, etc. interoperate rather than requiring a single-vendor SDK everywhere.
- **Instrumentation**: automatic (agents/SDKs that patch common frameworks — HTTP clients, DB drivers) plus manual (custom spans around business logic).
- **Exporters and the Collector**: the OTel Collector receives telemetry (OTLP protocol) from instrumented services, batches/filters/samples it, and exports to a backend (Prometheus, Jaeger, Tempo, a SaaS vendor) — decoupling instrumentation from backend choice.
- **Sampling**: head-based (decide to sample at trace start, e.g. 1% of traces) vs tail-based (buffer spans and decide after seeing the full trace, so you can always keep error/slow traces even at low overall sample rate). Tail-based sampling is essential at Facebook-Live-Comments scale, where head sampling at low rates would likely miss rare but important slow traces.

### Prometheus + Grafana for Dashboards and Alerting

Prometheus is a pull-based metrics system: it scrapes `/metrics` HTTP endpoints on a configured interval, stores time series in its own TSDB, and evaluates alerting/recording rules against **PromQL**. Core primitives: `counter` (monotonically increasing, e.g. total requests), `gauge` (can go up/down, e.g. queue depth), `histogram` (bucketed observations enabling quantile estimates like p99 latency), `summary` (client-side quantiles, less flexible for aggregation across instances than histograms).

**Alertmanager** deduplicates, groups, and routes alerts fired by Prometheus rules (e.g., `histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m])) > 0.5`) to Slack/PagerDuty/email, with silencing and inhibition rules to reduce noise. **Grafana** is the visualization layer on top — dashboards built from PromQL (or other data source) queries, with panels, alerting UI, and annotations (e.g., marking deploy events on a latency graph so a regression can be visually correlated with a specific release).

Best practice at scale: the **RED method** for request-driven services (Rate, Errors, Duration) and the **USE method** for resources (Utilization, Saturation, Errors), plus Google's **four golden signals** (latency, traffic, errors, saturation) as the baseline dashboard for any service.

### SLI, SLO, SLA — Definitions and Relationships

These three terms form a strict hierarchy, as defined in Google's SRE book:

- **SLI (Service Level Indicator)**: a *quantitative measurement* of some aspect of the service — e.g., "the proportion of HTTP requests completed in under 300ms" or "the fraction of comments delivered to viewers within 2 seconds of posting." It is a ratio: good events / valid events.
- **SLO (Service Level Objective)**: a *target value or range* for an SLI over a time window — e.g., "99.9% of comment-delivery requests complete in under 2 seconds, measured over a rolling 28 days." SLOs are internal engineering targets, chosen to be achievable and meaningfully tied to user happiness (not 100% — perfection is economically irrational and masks the cost of reliability work).
- **SLA (Service Level Agreement)**: an *external, often contractual* promise to customers, usually with a penalty (credits, refunds) for violation, typically set looser than the internal SLO so there is margin for error before a business consequence is triggered (e.g., SLO 99.9%, SLA 99.5%).

The relationship: SLI is the measurement, SLO is the internal bar you hold yourself to, SLA is what you promise the outside world — with the SLO acting as an early-warning buffer before an SLA breach becomes a financial/contractual event.

### Error Budgets and Release Gating

An **error budget** is the complement of the SLO: if the SLO is 99.9% success over 28 days, the error budget is the allowed 0.1% of "bad" events in that window. This budget is a shared resource between product/feature velocity and reliability: as long as budget remains, teams can ship features, run experiments, and take risks (canary rollouts, config changes); once the budget is exhausted, the SRE-driven policy typically freezes further risky releases and reallocates engineering effort to reliability work until the error rate recovers below the objective. This converts an abstract "be reliable" mandate into an objective, data-driven, non-political gate — release decisions stop being a debate and become a budget-remaining check, exactly analogous to a financial budget.

### The Post-Mortem Process (Blameless Postmortems)

A blameless postmortem is written after any significant incident, with the explicit premise that people make the best decisions they can with the information available at the time, and that the goal is to find systemic and process causes, not to assign individual blame. This matters practically: engineers who fear blame hide information, omit details, or route around admitting mistakes, which degrades the quality of the incident record and repeats the failure later. A good postmortem includes: a timeline (detection, diagnosis, mitigation, resolution timestamps), impact (users/revenue/SLO burn affected), root cause(s) — often multiple contributing factors, not one — what went well/poorly/was lucky, and a concrete, owned, tracked list of follow-up action items (not just "be more careful"). Postmortems should be reviewed in a regular forum and their action items tracked to closure like any other engineering work, or the practice degrades into theater.

### FinOps Basics

FinOps is the discipline of bringing financial accountability to variable cloud spend, treating cost as a first-class engineering signal alongside latency and error rate. Core practices: tagging/labeling every resource by team/service/environment for cost attribution; unit economics (cost per request, cost per active user, cost per GB ingested) rather than raw total spend, so cost scales meaningfully are visible; rightsizing and autoscaling to avoid paying for idle capacity; commitment discounts (reserved instances/savings plans) for predictable baseline load, on-demand/spot for bursty or interruptible load; and — directly relevant to this module — treating **observability data volume itself as a cost driver**: high-cardinality metrics, unsampled traces, and verbose logs at massive scale can become one of the largest line items in a cloud bill, so sampling strategy, log retention tiers (hot/warm/cold), and cardinality limits are as much a cost decision as a technical one.

---

## Case Study Solution: Facebook Live Comments

### Problem Statement & Clarifying Requirements

Design the comment stream for Facebook Live: viewers post text comments while watching a live broadcast, and every other viewer sees new comments appear in near real time, ordered coherently, without the video experience stalling.

**Functional requirements**
- Users can post a comment on a live video.
- All viewers of that video receive new comments in (approximately) real time.
- Comments are durably stored and remain readable after the broadcast ends (VOD replay).
- Basic moderation/spam filtering before a comment fans out.

**Non-functional requirements**
- Extremely high write rate: a single viral live stream (e.g., a major creator or breaking news event) can draw tens of millions of concurrent viewers, with comment rates spiking by 10-100x within seconds of a notable moment in the video.
- Very high fan-out multiplier: each comment must be delivered to potentially millions of concurrent viewers of the same broadcast — this is a one-to-many fan-out problem, not a simple CRUD write path.
- Low end-to-end latency: sub-second to low-seconds delivery so comments feel "live."
- Must detect a delivery/latency regression within seconds to tens of seconds during an active broadcast — a slow rollout of a bad build during a viral event is a visible, embarrassing outage, so observability must be near-real-time, not just next-day dashboards.
- Availability favored over strict consistency: acceptable to drop or delay a rare comment under extreme load; unacceptable to crash the video player or block video playback.

### Capacity Estimation

Assume a viral live event with 10 million concurrent viewers, and roughly 1% of viewers commenting during a peak moment, each averaging one comment per 30 seconds during that peak:

- Peak commenters: 10,000,000 × 1% = 100,000 concurrent commenters.
- Write rate: 100,000 / 30s ≈ **3,300 comments/sec** at peak for a single hot broadcast (real large-scale events have reported comment rates in the low thousands per second range during peak moments, consistent with this order of magnitude).
- Fan-out multiplier: each of those ~3,300 comments/sec must be delivered to up to 10,000,000 viewers of that same video, i.e., a **fan-out ratio of ~3,000:1 to 10,000:1** depending on how many viewers are actively "subscribed" to the comment stream vs. just watching video with comments collapsed.
- Effective fan-out throughput: 3,300 comments/sec × 10,000,000 viewers, if delivered naively as a full fan-out-on-write, is on the order of **10^10 delivery events/sec** for one broadcast — clearly infeasible to push individually; this forces a broadcast/pub-sub delivery model (fan-out-on-read via a shared topic, e.g. per-video pub-sub channel with client push over WebSocket/long-poll) rather than per-follower fan-out-on-write, which is standard for a single shared object like a live video (as opposed to a personalized feed).
- Storage: 3,300 comments/sec × ~200 bytes/comment ≈ 660 KB/sec ≈ 57 GB/day for one hot broadcast; trivial to store, the bottleneck is fan-out delivery and read amplification, not storage.

### High-Level Architecture

```
 Client (web/mobile)
     │  POST /videos/{id}/comments   (comment write)
     │  WS/long-poll subscribe        (comment read stream)
     ▼
 ┌─────────────────┐        ┌──────────────────────┐
 │  Edge / API GW   │──────▶│  Comment Ingestion    │
 │ (assigns trace_id│        │  Service (validate,   │
 │  = correlation id│        │  moderate, persist)   │
 └─────────────────┘        └──────────┬────────────┘
                                        │ append
                                        ▼
                              ┌──────────────────┐
                              │  Durable Log /     │
                              │  Comment Store      │
                              │  (Kafka topic per   │
                              │   video-shard + DB) │
                              └──────────┬────────┘
                                        │ consume
                                        ▼
                              ┌──────────────────┐
                              │  Fan-out / Broadcast│
                              │  Service (per-video  │
                              │  pub-sub topic,      │
                              │  edge push servers)  │
                              └──────────┬────────┘
                                        │ push
                     ┌──────────────────┼──────────────────┐
                     ▼                  ▼                  ▼
              Viewer WS conn 1   Viewer WS conn 2 ...  Viewer WS conn N
                     (millions of persistent connections, geo-sharded)

 ── Observability plane (side-car / async, all components emit into it) ──
     Metrics ─▶ Prometheus (scrape) ─▶ Grafana dashboards + Alertmanager
     Traces  ─▶ OTel SDK ─▶ OTel Collector ─▶ Jaeger/Tempo (tail-sampled)
     Logs    ─▶ structured JSON w/ trace_id ─▶ Loki/ELK
     All three correlated by trace_id generated at Edge/API GW
```

Ingestion is decoupled from fan-out by a durable log (Kafka, partitioned by video ID) so a burst of writes can be buffered and fan-out consumers can scale independently and replay on failure. Fan-out servers hold the actual WebSocket connections, sharded by video ID and geography so all viewers of one video map to a bounded set of fan-out nodes rather than every node holding every video's state.

### API Design

```
POST /v1/videos/{video_id}/comments
Headers: Authorization, traceparent (W3C trace context, or generated server-side)
Body: { "text": "great goal!!", "client_ts": 1735900000123 }
Response 202 Accepted: { "comment_id": "...", "status": "queued" }
```
Asynchronous 202 response — the comment is durably queued, not necessarily fanned out yet, which decouples perceived write latency from fan-out latency.

```
Stream (WebSocket): wss://live.example.com/v1/videos/{video_id}/comments/stream
Server → Client frames:
{ "type": "comment", "comment_id": "...", "author_id": "...",
  "text": "...", "server_ts": 1735900000456, "seq": 88213 }
{ "type": "heartbeat", "server_ts": ... }
```
A monotonic per-video `seq` number lets clients detect gaps (dropped frames under load) and optionally request a backfill via a cheap REST fallback (`GET /v1/videos/{id}/comments?after_seq=...`), which is also the graceful-degradation path for clients on flaky connections or when a fan-out node briefly falls behind (favoring availability/eventual delivery over blocking).

### Data Model

```
Comment {
  comment_id: string (ULID — sortable + unique)
  video_id: string
  author_id: string
  text: string (validated/truncated, e.g. <= 500 chars)
  created_at: timestamp (server-assigned, authoritative)
  client_ts: timestamp (client-claimed, for latency measurement only)
  seq: int64 (monotonic per video_id, assigned at ingestion)
  moderation_status: enum(pending, approved, hidden)
  trace_id: string (correlation id for the write path)
}
```
Sharded/partitioned by `video_id` (hot videos may need sub-sharding by `video_id + hash(author_id)` if a single viral video's write rate exceeds one partition's throughput).

**Telemetry emitted at each stage** (all tagged with `trace_id`, `video_id`):
- Ingestion: `comment_ingest_latency_ms` (histogram), `comment_ingest_errors_total` (counter, by error type), `moderation_reject_total`.
- Durable log: `kafka_producer_ack_latency_ms`, `kafka_consumer_lag` (gauge, per partition — critical early-warning signal for fan-out falling behind).
- Fan-out: `fanout_latency_ms` (histogram: time from `created_at` to push to each connection), `fanout_delivered_total` / `fanout_dropped_total`, `active_ws_connections` (gauge, per video/shard).
- Client-observable (via RUM/beacon): end-to-end `comment_e2e_latency_ms` = client receive time − `created_at`, the SLI closest to actual user experience.

### Deep Dive: SLIs/SLOs, Error Budget Gating, and Tracing a Regression

**Concrete SLIs/SLOs for this system:**

| SLI | Definition | SLO |
|---|---|---|
| Comment write availability | % of POST /comments returning 2xx | 99.95% over 28 days |
| Comment ingest latency | p99 time from request received to durably queued | ≤ 300ms |
| Fan-out delivery latency (p99) | time from `created_at` to delivery at a viewer's WS connection | ≤ 2s |
| Fan-out delivery latency (p50) | median delivery latency | ≤ 500ms |
| Fan-out success rate | delivered_events / (delivered_events + dropped_events) per video, per minute window | ≥ 99.9% |
| Trace/log completeness | % of requests with a resolvable trace_id across all hops | ≥ 99.99% (an observability SLI on the observability system itself) |

**Error budget gating a risky rollout:** Suppose the 28-day fan-out-success-rate SLO is 99.9% (budget: 0.1% of comment-deliveries may fail). A new fan-out server build is ready to deploy during an ongoing high-traffic live event (e.g., a major sports final). The release process checks current error-budget burn: if the trailing-window burn rate is already elevated (say, 40% of the 28-day budget consumed in the last 24 hours due to an unrelated blip), the deploy is automatically gated/blocked by policy, and the rollout is deferred until after the event or done as a 1% canary with an automatic rollback trigger tied to a **fast-burn alert** (e.g., "if fan-out-success-rate drops below 99.5% over any 5-minute window, roll back automatically") rather than a full blue/green cutover mid-event. This is exactly the SRE error-budget policy pattern: the budget is what turns "should we deploy right now?" from a judgment call into a rule the on-call engineer (or an automated release pipeline) can apply without escalation.

**Using distributed tracing + correlation IDs to debug a specific regression:** Imagine Grafana's dashboard shows p99 fan-out latency jump from 800ms to 4.5s starting at 20:14 UTC, correlated with a deploy annotation at 20:12. The on-call engineer:
1. Uses the annotation to identify the suspect deploy (fan-out service v482).
2. Filters Jaeger/Tempo for traces with `fanout_latency_ms > 2000` and `service=fanout` in the 20:12–20:20 window (tail-based sampling ensures these slow-outlier traces were kept even though overall sampling might be 1%).
3. Opens several such traces: each shows the full span tree — ingestion span (normal, ~50ms), Kafka produce span (normal), then a fan-out span with an unusually long child span calling a new "author reputation" lookup service that was added in v482 and is doing a synchronous, unbatched RPC per comment per shard.
4. Pivots from the trace's `trace_id` directly into Loki/ELK, pulling every structured log line tagged with that `trace_id` across ingestion, Kafka consumer, and fan-out — confirming the reputation-lookup calls are timing out under the shard's connection pool limit at high concurrency.
5. Root cause identified within minutes (not hours) specifically because the correlation ID tied one user-visible slow comment to its exact spans and log lines across three separate services, instead of engineers manually eyeballing timestamps across three separate log stores.
6. Mitigation: roll back v482 (or feature-flag off the reputation lookup) — the error budget burn from this incident is logged and reduces the remaining budget for the rest of the 28-day window, informing whether further risky changes are permitted before the window resets.

### Trade-offs and Alternatives Considered

- **Fan-out-on-write vs. fan-out-on-read**: personalized feeds (e.g., News Feed) typically fan out on write to each follower's inbox; a live video's comment stream is naturally fan-out-on-read from one shared per-video topic, since all viewers want the identical stream — this avoids the O(comments × followers) write amplification a per-user inbox model would create.
- **WebSockets vs. long-polling vs. SSE**: WebSockets give the lowest latency and best throughput per connection but need sticky connection management and graceful draining during deploys; long-polling/SSE are simpler and more firewall-friendly but add per-poll overhead at this scale. Large-scale systems typically use WebSockets with a fallback to long-polling for constrained clients.
- **Strong ordering vs. availability**: guaranteeing global strict ordering of comments across a highly sharded fan-out layer is expensive; the design accepts a monotonic per-shard `seq` and eventual, mostly-ordered delivery, favoring availability and low latency — consistent with the non-functional requirement that a dropped/delayed rare comment is acceptable but blocking video is not.
- **Head sampling vs. tail sampling for traces**: head sampling is cheaper but can miss the rare slow trace that matters most during an incident; tail sampling costs more (buffering all spans until a trace completes) but is necessary here specifically because regression detection speed is a hard requirement.
- **Full unsampled tracing at 10M concurrent viewers**: cost-prohibitive; the design samples client-side "comment received" beacons and server spans, relying on aggregate metrics (Prometheus) for volume-wide SLO tracking and using sampled/tail-sampled traces only for root-cause drill-down, not for every event.

### How Real Systems Solve This

Meta's own engineering blog describes the original "Live Commenting" system (2011) as built to handle real-time comment/like fan-out on high-traffic posts using a publish-subscribe model with dedicated infrastructure separate from the main News Feed write path, explicitly to avoid overloading per-user fan-out systems with a single hot object's traffic — the same fan-out-on-read principle used above. System-design write-ups of the modern "Facebook Live Comments" problem (Hello Interview, GeeksforGeeks, systemdesign.one) converge on the same core shape: an ingestion/validation tier, a durable append log partitioned by video/room, and a broadcast tier holding WebSocket fan-out separate from ingestion so the two scale independently — validating the architecture in this module. Broadly across the industry, high-scale observability for this class of system follows the pattern of RED-method dashboards for the request-facing tiers, dedicated consumer-lag and connection-count gauges for the streaming tiers, and tail-based-sampled distributed tracing (via OpenTelemetry-compatible collectors) specifically so a rare, severe regression during a viral spike is caught by an automated alert within seconds rather than discovered from user complaints.

---

## Sources

- [Google SRE Book — Service Level Objectives](https://sre.google/sre-book/service-level-objectives/)
- [Google SRE Workbook — Implementing SLOs](https://sre.google/workbook/implementing-slos/)
- [Google SRE Workbook — Error Budget Policy](https://sre.google/workbook/error-budget-policy/)
- [OpenTelemetry Context Propagation: W3C Trace Context and Baggage — Uptrace](https://uptrace.dev/opentelemetry/context-propagation)
- [W3C Trace Context Explained: Traceparent & Tracestate — Dash0](https://www.dash0.com/knowledge/w3c-trace-context-traceparent-tracestate)
- [Understanding OpenTelemetry Traces and Spans](https://oneuptime.com/blog/post/2026-02-20-opentelemetry-traces-spans-guide/view)
- [Live Commenting: Behind the Scenes — Engineering at Meta](https://engineering.fb.com/2011/02/07/core-infra/live-commenting-behind-the-scenes/)
- [Facebook Live Comments System Design Interview Guide — Hello Interview](https://www.hellointerview.com/learn/system-design/answer-keys/fb-live-comments)
- [Design Facebook's Live Comments System — Hello Interview](https://www.hellointerview.com/learn/system-design/problem-breakdowns/fb-live-comments)
- [Live Comment System Design — systemdesign.one](https://systemdesign.one/live-comment-system-design/)
- [Design Facebook's live update of comments on posts — GeeksforGeeks](https://www.geeksforgeeks.org/system-design/design-facebooks-live-update-of-comments-on-posts-system-design/)
- [A Complete Guide to Error Budgets — Nobl9](https://www.nobl9.com/resources/a-complete-guide-to-error-budgets-setting-up-slos-slis-and-slas-to-maintain-reliability)
