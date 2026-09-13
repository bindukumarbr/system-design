# Case Study: Facebook Live Comments (Observability)
- **Requirements:** Diagnose latency spikes during high-traffic live events.
- **Architecture:** W3C Traceparent headers passed across all microservices (OpenTelemetry).
- **Key Insight:** Don't trace every request (too expensive). Use Tail-Based Sampling to keep 100% of traces for requests that fail or exceed P99 latency, dropping the "normal" ones.
