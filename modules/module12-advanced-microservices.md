# Module 12: Advanced Microservices

## Core Concepts

### Circuit Breaker Pattern

A circuit breaker wraps a remote call and stops making it once failures cross a threshold, protecting the caller from wasting threads/connections on a struggling callee, and protecting the callee from being hammered while it recovers. Netflix's Hystrix popularized this for JVM microservices; it's now largely superseded by **resilience4j**, but the state machine is the same across implementations and mesh sidecars (Envoy's outlier detection applies the same idea to connection pools).

Three states:
- **Closed** — requests flow normally. The breaker tracks success/failure/timeout outcomes over a rolling window (time- or count-based). Once the failure/slow-call rate exceeds a threshold (e.g. 50%) over a **minimum call volume** (so 2-of-3 failures on a low-traffic service doesn't trip it), it moves to Open.
- **Open** — all calls fail immediately, fail-fast, without touching the network. This is the core value: it stops cascading failure, where one slow dependency exhausts the caller's thread/connection pool, which then makes the caller slow to *its* callers, propagating up the whole call graph — the exact failure mode that motivated Hystrix. Stays open for a configured wait duration (e.g. 30–60s).
- **Half-Open** — after the wait, a small number of trial requests are let through. Enough successes returns to Closed; failures send it back to Open, often with backoff on repeated re-trips.

Implementation details worth knowing: track "slow call" as a distinct failure mode from exceptions (a 30s hang is often worse than a fast failure); scope breakers **per downstream dependency**, not globally — Hystrix enforced this with per-dependency thread pools (the **bulkhead pattern**: isolating resource pools per dependency so one slow dependency can't starve others).

### Retry with Exponential Backoff and Jitter

Retries handle transient errors, but naive immediate retries make things worse: if many clients fail simultaneously and all retry immediately, they recreate the spike that caused the failure — a **retry storm** / thundering herd.

**Exponential backoff** spaces retries out with growing delays (`delay = base * 2^attempt`, capped). This reduces average load but does *not* by itself fix synchronization: if 1,000 clients fail at t=0, they all retry together at t=100ms, then all together at t=300ms — locked in phase.

**Jitter** is what breaks the synchronization: each client picks a randomized delay (e.g. "full jitter" `random(0, base*2^attempt)`, per AWS's widely cited guidance) instead of a deterministic one. The precise point to make in an interview: **jitter's job is to decorrelate independent clients' retry times so aggregate retry load smooths into a trickle instead of synchronized spikes** — exponential growth alone reduces retry *rate* but preserves *synchronization*; jitter removes the synchronization itself. Most production clients (AWS SDK, gRPC, resilience4j's randomized backoff) default to jitter for this reason. Retries should be bounded (max attempts, overall deadline), paired with a circuit breaker so clients stop retrying an already-tripped dependency, and restricted to idempotent operations (or use idempotency keys).

### Strangler Fig Pattern

Named for the vine that grows around a host tree and eventually replaces it: a routing layer (reverse proxy, gateway, or flagged code path) sits in front of a monolith, and new or re-implemented functionality is built as separate services while the router sends matching traffic to them and everything else still hits the monolith. Traffic is peeled off route by route until the monolith is empty and can be retired. This bounds risk (each slice is independently deployable/revertible) versus a big-bang rewrite. The hard part is usually **shared data** — if the new service owns "orders" but the monolith's DB still references them, you need a migration strategy (dual writes, CDC pipelines, an anti-corruption layer) and must tolerate eventual consistency during the transition.

### Sidecar Pattern

A sidecar is a second process/container in the same pod, sharing the network namespace so it can transparently intercept traffic, handling cross-cutting concerns without app code changes:
- **Logging** — an agent (e.g. Fluent Bit) ships stdout/logs centrally.
- **Monitoring** — a metrics exporter standardizes telemetry (e.g. Prometheus) regardless of app language.
- **Auth/mTLS proxies** — a proxy (Envoy) terminates mTLS and enforces authn/authz, forwarding only verified plaintext to the app on localhost.
- **Rate limiting/traffic shaping** — enforced before the request reaches app code (see the case study below).

Benefit: uniform, battle-tested implementations injected by the platform regardless of app language. Cost: per-pod resource overhead and an extra (sub-millisecond, but non-zero) network hop.

### Service Mesh (Istio, Linkerd)

A service mesh applies the sidecar pattern fleet-wide plus a control plane that centrally configures every sidecar ("data plane"). **Istio** uses Envoy as its sidecar and a control plane (istiod) pushing routing, TLS, retry/circuit-breaker, and rate-limit config to every proxy. **Linkerd** uses its own lightweight Rust proxy, trading Istio's larger feature surface for simplicity and lower resource cost. Problems a mesh solves that are painful per-service: automated mTLS/cert rotation everywhere; uniform retry/timeout/circuit-breaker policy configured declaratively instead of re-implemented per service; weighted traffic shifting for canaries/blue-green without app-level routing code; consistent golden-signal metrics and tracing propagation, since every hop passes through a proxy; and zero-trust authorization enforced at the network layer. Trade-off: added latency/resource overhead per hop and a control plane to operate — usually justified once an org has enough services that per-service reimplementation becomes the bigger cost.

### Saga Pattern (Revisited)

For a transaction spanning services (e.g. reserve inventory, charge payment, schedule shipping — each owned by a different service/DB), distributed 2PC is usually rejected because cross-service locking hurts availability and throughput. A **saga** runs a sequence of local transactions, each publishing an event that triggers the next; on failure, completed steps are undone via **compensating transactions** rather than an atomic rollback. Sagas are **choreographed** (services react to each other's events — simple for few steps, hard to reason about as steps grow) or **orchestrated** (a coordinator explicitly calls each step and issues compensations — easier to observe, adds a component). Trade-off versus 2PC: give up strong consistency (partial state is visible during execution) for availability and service autonomy; compensations must be designed per step, not just inverted (you can't un-send an email — compensate with a correction instead).

### Canary Releases and Feature Flags

A **canary release** ships a new version to a small slice of infrastructure/traffic, watches error rate/latency/business metrics, then ramps up — or rolls back instantly by shifting weight to zero. This bounds blast radius versus deploying to everyone at once. **Feature flags** decouple deployment from activation: code ships dark (flag off), gets verified, then is turned on for a small percentage and ramped, fully reversible without a redeploy — also enabling targeted rollout and A/B testing. Combined, canary de-risks the infrastructure change and flags de-risk the behavioral change; both are essential for safely rolling out something like a new rate-limit threshold, discussed next.

---

## Case Study Solution: Distributed Rate Limiter

### Problem Statement & Requirements

Design a system that protects a fleet of stateless, horizontally-scaled API services from abuse and enforces usage quotas.

**Functional**: limit per API key / user ID / IP; support multiple plan tiers and simultaneous windows (e.g. 100/min *and* 10,000/day); return `429` with `Retry-After` and `X-RateLimit-*` headers; limits updatable dynamically without redeploys.

**Non-functional**: **global correctness** across many instances (100 req/min must not become 5,000/min across 50 instances — rules out purely per-instance in-memory counters as the sole mechanism); **low added latency** (single-digit ms at p99, hot path); **availability over precision** (a limiter outage must never become an API outage); **horizontal scalability** to 100k+ checks/sec; **low memory** relative to millions of keys.

### Capacity Estimation

500,000 active API keys, 50,000 req/sec steady state, bursting to 200,000 req/sec — each needing one check-and-increment. A single-threaded Redis instance handles 100k–200k simple ops/sec, and an O(1) Lua-scripted check-and-increment keeps per-op cost flat, so a small Redis Cluster (3–6 shards) clears this with headroom. Working set: ~500k keys × ~3 windows × ~100 bytes ≈ low hundreds of MB, fits in memory. Same-AZ round trip to Redis is ~0.3–1ms, the dominant cost and within budget; cross-region calls would blow it, so keep a regional cluster per deployment region.

### High-Level Architecture

Three placement options: (1) in-process per-instance — fast but no cross-fleet correctness, rejected as sole mechanism; (2) sidecar/gateway plugin calling a shared store; (3) fully centralized limiter service via RPC. Option (2) wins: it reuses the sidecar/mesh infrastructure already terminating mTLS and auth, keeps decision logic local, and only the store lookup crosses the network.

```
                 ┌───────────────────────────────┐
                 │          Redis Cluster          │
                 │  (sharded by rate-limit key)     │
                 │  token-bucket state per key       │
                 └───────────▲──────────▲──────────┘
                     check&incr        check&incr
                     (Lua script)      (Lua script)
          ┌───────────────┴──┐      ┌───┴───────────────┐
          │ Envoy sidecar (A) │      │ Envoy sidecar (B)  │
          │ mTLS + rate limit │      │ mTLS + rate limit  │
          └───────────▲───────┘      └────────▲───────────┘
                       │                       │
                 ┌─────┴────┐            ┌─────┴────┐
                 │ Service A│  ... (xN)  │ Service B│
                 └──────────┘            └──────────┘
                       ▲
                 ┌─────┴─────┐
                 │  Clients   │
                 └───────────┘

  Config plane pushes {key-pattern -> limit, window} to all
  sidecars, independent of app deploys (feature-flagged rollout).
```

Without a mesh, the same logic is a plugin in a shared API gateway (Kong/NGINX modules, AWS API Gateway usage plans) — architecturally equivalent, centralized instead of colocated.

### API Design

```
CheckAndIncrement(
  key: string,          // "apikey:abc123:route:/v1/charges"
  limit: int, windowSeconds: int, cost: int = 1
) -> { allowed: bool, remaining: int, resetAtEpochSeconds: int }
```

`allowed=false` → gateway returns `429` with `Retry-After` and `X-RateLimit-*` headers without forwarding the request; `allowed=true` → request proceeds, response carries the same headers. Multiple simultaneous windows are multiple key checks (or one script checking several); any window exceeded denies the request.

### Data Model: Token Bucket, and Why

Four candidates: **fixed window** (trivial, O(1), but a boundary bug lets `2x limit` through across a window edge); **sliding log** (perfectly precise, but memory grows with request volume — expensive at scale); **sliding window counter** (weights current+previous fixed windows — Cloudflare's published production approach, near sliding-log accuracy at near fixed-window cost); **token bucket** (per-key bucket refilling at a steady rate, up to a burst cap; leaky bucket is its queueing inverse, better for traffic shaping than rejection-based limiting).

**Chosen: token bucket**, stored as `{tokens, last_refill_ts}` per key, mutated atomically in a Lua script. Justification: it naturally expresses both a sustained rate *and* an allowed burst in one primitive — the actual shape of real API quotas (matches how payment APIs like Stripe publicly describe limits) — needs only two fields per key (O(keys), not O(request volume) like sliding log), and refill is cheap lazy arithmetic (`tokens = min(burst, tokens + elapsed*rate)`) so idle keys cost nothing between requests. Its one imprecision versus the sliding-window counter — allowing a full burst right after a refill — is a feature here, since real clients need burst headroom. The Lua script performs the read-modify-write atomically so concurrent gateway instances never race on the same key.

### Deep Dive: Sidecar and Circuit-Breaker Ties; Canary Rollout

**Sidecar tie-in**: the limiter is the sidecar pattern applied to a new concern — the same Envoy proxy already terminating mTLS/auth hosts a rate-limit filter (Envoy's `ratelimit` filter is the real-world instance) that calls the shared store before the request reaches app code. Service teams write zero rate-limiting code; the platform owns it once.

**Circuit breaker tie-in — the key design decision**: Redis is now a hot-path dependency for every request fleet-wide, so its own outage risks becoming the API's outage — the exact cascading-failure scenario breakers exist for. Wrap sidecar→Redis calls in a circuit breaker; on trip, the sidecar stops calling Redis for a cool-down. The real decision is **what happens while open — fail open vs. fail closed**:
- **Fail open** (let requests through unmetered) is the right default: a brief unmetered window is far less harmful than a rate-limiter outage taking down the entire API — this is the standard trade-off in production write-ups (e.g. Figma's public post on their limiter). Treat the limiter as best-effort protection, not a strict security boundary.
- **Fail closed** only for hard business/security limits (e.g. fraud control), and even then usually paired with a coarse local in-memory backstop (e.g. N req/sec per instance) rather than pure deny-all.

A short-TTL local cache of "definitely allowed" decisions also cuts store calls for the common case, so a Redis blip degrades throughput gracefully rather than correctness.

**Canary rollout via feature flags**: a stricter limit pushed fleet-wide in one step could 429 legitimate traffic suddenly. New thresholds are gated through the same config-plane push as a percentage rollout — 1% of keys or one region first, watch 429/error rates, ramp to 10/50/100%, with instant rollback by reverting the flag.

### Trade-offs and Alternatives Considered

- **Centralized gRPC limiter service** vs. sidecar-to-store: cleaner, language-agnostic API, but adds a full extra hop versus local-check; chosen design accepts Lua-script coupling for lower latency.
- **Sharding by key**: good locality, no cross-shard coordination, but a very hot key can hot-spot one shard — mitigated with consistent hashing and, for the largest keys, approximate sub-counters summed across shards.
- **Local per-instance limiting**: rejected as sole mechanism, kept as a pre-filter — obviously abusive traffic (e.g. 10k req/sec from one caller) is rejected locally before consulting Redis.
- **Sliding log precision** rejected for memory cost; token bucket's minor over-permissiveness at refill boundaries accepted as a deliberate trade for O(1) memory and burst support.

### How Real Systems Solve This

**Stripe** runs rate limiting in multiple tiers (request-based, concurrent-request, and fair-usage/anti-abuse layers) rather than one global counter, isolating noisy customers while staying highly available for payments traffic. **Cloudflare** publishes an approximation of the sliding-window algorithm — weighting current and previous fixed windows — as the pragmatic middle ground between fixed-window's boundary bug and sliding-log's memory cost, at edge scale. **Figma** has written about moving from per-endpoint fixed windows toward a centralized, cost-weighted scheme as their API surface and abuse patterns grew. **Redis** documents reference Lua implementations of fixed window, sliding window, token bucket, leaky bucket, and GCRA — the same primitives this design builds on.

## Sources

- [How it Works — Netflix/Hystrix Wiki](https://github.com/netflix/hystrix/wiki/how-it-works)
- [resilience4j — Comparison to Netflix Hystrix](https://resilience4j.readme.io/docs/comparison-to-netflix-hystrix)
- [Exponential Backoff and Jitter / Retry Storms explained](https://web-alert.io/blog/retry-storms-exponential-backoff-jitter-explained)
- [Rate Limiter — Sliding Window Counter](https://medium.com/@avocadi/rate-limiter-sliding-window-counter-7ec08dbe21d6)
- [Deep Diving into Cloudflare's Rate Limiting Architecture](https://medium.com/@gsoumyadip2307/deep-diving-into-cloudflares-rate-limiting-architecture-7a5fc521ffd3)
- [Cloudflare's approximation of the moving window algorithm — discussion](https://github.com/alisaifee/limits/discussions/245)
- [An alternative approach to rate limiting — Figma Blog](https://www.figma.com/blog/an-alternative-approach-to-rate-limiting/)
- [Build 5 Rate Limiters with Redis: Algorithm Comparison Guide](https://redis.io/tutorials/howtos/ratelimiting/)
- [Stay within limits: API rate-limit-friendly pattern for Stripe webhooks](https://stripe.dev/blog/stay-within-limits-api-rate-limit-friendly-pattern-for-stripe-webhooks)
- [Inside Stripe's Rate Limiter Architecture (video breakdown)](https://www.youtube.com/watch?v=Oxy-6MAiYPw)
- [This Is How Stripe Does Rate Limiting to Build Scalable APIs](https://newsletter.systemdesign.one/p/rate-limiter)
