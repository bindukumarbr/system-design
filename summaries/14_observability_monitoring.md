# Module 14: Observability & Monitoring

## Core Concepts

### The Three Pillars of Observability
1. **Metrics:** Time-series numbers (Counters, Gauges, Histograms). Cheap to store. Perfect for dashboards and alerts (e.g., "p99 latency is up"). *Blind spot: lacks context on specific requests.*
2. **Logs:** Rich text events (`{"level": "error", "msg": "timeout"}`). Excellent for debugging. *Blind spot: expensive to store and slow to search at high volume.*
3. **Traces:** Tracks a single request as it jumps across microservices. Built using a tree of **Spans**. *Blind spot: 100% sampling is too expensive.*

### Correlation IDs (The Glue)
- The API Gateway generates a `trace_id` for the incoming request and passes it to every microservice via HTTP headers.
- Every metric, log line, and trace span must include this `trace_id`. When a request fails, you can pull exactly the logs related to it across 10 different microservices.

### SLI, SLO, SLA
- **SLI (Indicator):** The metric (e.g., "% of HTTP 200s in < 300ms").
- **SLO (Objective):** Internal team goal (e.g., "99.9% over 28 days").
- **SLA (Agreement):** Contract with customers (e.g., "99.5%, or we pay refunds").

### Error Budgets
- 100% minus the SLO. If SLO is 99.9%, the Error Budget is 0.1%.
- If you have budget left, you can deploy features fast. If the budget is exhausted, deploying is frozen and all engineers must fix reliability.

---

## Case Study: Facebook Live Comments
- **Requirements:** Users post comments on live video. Millions of viewers must see them in real time.
- **Estimations:** 10M concurrent viewers on a viral stream. 3,300 comments/sec. **Fan-out ratio:** 3,300 comments * 10M viewers = billions of websocket pushes/sec.
- **Architecture:** 
  - **Ingestion Service:** Receives comment, durably saves to Kafka topic for the video.
  - **Fan-Out Service:** WebSocket servers read from Kafka and push to millions of connected viewers.
  - **Telemetry:** W3C Trace Context headers passed along. Tail-based sampling is used so slow traces are kept even if overall sampling is low.
- **Aha! Insights:**
  - **Tail-based Sampling vs Head-based Sampling:** If you sample 1% of requests at the start (Head), you will likely miss the rare slow trace that caused a timeout. Buffering traces and keeping the 1% that are slow/errors (Tail) is much better for debugging viral spikes.
  - **Fan-Out-On-Read:** Instead of writing the comment to 10M separate follower inboxes, write it to 1 Kafka topic, and have the 10M viewers pull/listen to that topic.
