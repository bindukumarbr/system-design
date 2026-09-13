# Module 12: Advanced Microservices

## Core Concepts

### Circuit Breaker Pattern
- Prevents cascading failures. Wraps a remote call.
- **Closed:** Requests flow normally. If failure rate > 50%, state changes to Open.
- **Open:** All requests instantly fail without touching the network. This stops hammering a struggling downstream service and frees up caller threads.
- **Half-Open:** After 30s, let a few requests through to test if the service recovered.

### Retries with Exponential Backoff and Jitter
- **Exponential Backoff:** Spaces out retries (1s, 2s, 4s, 8s) to reduce load.
- **Jitter:** Adds randomness. **Crucial:** Without jitter, if 1000 clients fail at the exact same time, they will all retry at the exact same time (1s, 2s, 4s), creating synchronized retry storms. Jitter spreads them out into a smooth trickle.

### Sidecar Pattern & Service Mesh
- **Sidecar:** A helper container (e.g., Envoy) deployed alongside your app container in the same pod. It intercepts all network traffic.
- **Service Mesh (Istio/Linkerd):** A fleet of sidecars managed by a central Control Plane. It handles mTLS, retries, circuit breaking, and rate limiting transparently, so application devs don't have to write this logic.

### Strangler Fig Pattern
- Safely migrating from a monolith to microservices. Put an API Gateway in front of the monolith. Build a new microservice for "Billing". Tell the gateway to route `/billing` traffic to the new service. Gradually peel away routes until the monolith is dead.

---

## Case Study: Distributed Rate Limiter
- **Requirements:** Limit API requests per User ID / IP. Dynamic limits. High availability (limiter going down must not take down the API).
- **Estimations:** 200,000 requests/sec. Must be extremely low latency (p99 < 5ms).
- **Architecture:**
  - **Token Bucket Algorithm:** Kept in Redis as `{tokens, last_refill_timestamp}`. Excellent for allowing bursts of traffic while maintaining a sustained rate. Extremely memory efficient.
  - **Lua Scripts:** Atomically check-and-decrement tokens inside Redis so concurrent API servers don't face race conditions.
  - **Sidecar Envoy:** The rate limiter logic runs in an Envoy sidecar attached to every API instance. The sidecar intercepts the request, runs the Lua script in Redis, and either drops the request (429) or passes it to the app.
- **Aha! Insights:**
  - **Fail Open vs Fail Closed:** Wrap the sidecar-to-Redis call in a Circuit Breaker. If Redis goes down, what happens? **Fail Open.** It is far better to allow a few minutes of un-metered, free API usage than to take down the entire production API because the rate limiter crashed.
  - Token Bucket is strictly superior to Fixed Window (which suffers from boundary double-burst bugs) and Sliding Log (which consumes O(N) memory based on request volume).
