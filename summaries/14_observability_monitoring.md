# Fast-Track: Module 14 - Observability & Monitoring
**Core Concept:** Metrics, Logs, Traces. RED (Rate, Errors, Duration) / USE (Utilization, Saturation, Errors). SLI/SLO/SLA.
**Case Study:** Facebook Live Comments
- **Key Insight:** Tail-based sampling for tracing (keep traces of failed requests). Error budgets gate rollouts.
- **Takeaway:** Use Correlation IDs (W3C traceparent) across all microservice hops to debug bottlenecks.
