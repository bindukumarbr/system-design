# Module 22: Productionizing the URL Shortener

## Core Concepts

### Load Balancing & Traffic Distribution
- **Layer 7 Load Balancer:** Use ALB to route based on path (`/api/*` vs `/{code}`).
- **Least-Connections Algorithm:** Better than Round-Robin for redirects. A DB cache-miss stall shouldn't pile up on one instance.
- **Separate Target Groups:** Isolate Create API from Redirect API. A bulk import spike shouldn't starve the redirect threads.

### Auto-Scaling
- **Target Tracking:** Track `RequestCountPerTarget` or CPU. Better than step scaling for unpredictable viral traffic spikes.
- **Scale-out fast, scale-in slow:** 60s cooldown to scale up, 5m cooldown to scale down to avoid flapping.

### CDN Integration
- **Cache-Control:** If you must cache a permanent 301 redirect on a CDN, use an explicit `Cache-Control: max-age=30`. You MUST have an invalidation path if the link target changes.
- **CDN for TLS:** Terminate TLS at edge PoPs to cut connection latency even for 302 dynamic redirects.

### Rate Limiting (Abuse Prevention)
- **Token Bucket in Redis:** O(1) memory cost. Best for a public Create endpoint. Fall back to per-IP if unauthenticated. Tiered limits for paid users.

---

## Case Study: Surviving a Viral Link
- **Requirements:** Sustain an unexpected 50x traffic spike on a single shortened link.
- **Estimations:** Going from 300 req/min/instance to 15,000 req/min.
- **Architecture:** 
  - ALB detects target tracking threshold breach. ASG scales instances.
  - Redis absorbs the single hot-key read spike. Postgres load stays flat.
- **Aha! Insights:**
  - **Single Hot-Key Reads:** A viral link only hits ONE key. If that key is evicted, the sudden DB thundering herd can crash Postgres. Monitor Redis eviction rates as a leading indicator of DB failure.
  - **Target Tracking over Step Scaling:** Viral spikes are unpredictable. Target tracking automatically computes needed capacity, whereas step scaling requires predicting thresholds.
