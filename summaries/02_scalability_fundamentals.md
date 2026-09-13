# Module 02: Scalability Fundamentals

## Core Concepts

### Vertical vs. Horizontal Scaling
- **Vertical (Scale-up):** Adding more CPU/RAM to a single machine. 
  - *Pros:* No app changes needed. Good for stateful/legacy systems (e.g., primary relational DB).
  - *Cons:* Superlinear cost, hard ceiling, single point of failure.
- **Horizontal (Scale-out):** Adding more machines.
  - *Pros:* Linear cost, no architectural ceiling, handles failures gracefully (N+1 redundancy).
  - *Cons:* Requires stateless architecture, adds coordination overhead (load balancers, network).

### Statelessness (The Precondition for Horizontal Scale)
For a server to scale horizontally safely, it must be **stateless**: no request can depend on local memory/disk from a prior request.
- Push state to shared stores (Redis, Memcached) or to the client (JWTs).
- Keep local caches as disposable optimizations only.
- Design idempotent APIs.

### Load Balancing Algorithms
1. **Round-robin:** Fixed rotation. Cheap, but fails if request costs/durations vary wildly.
2. **Least connections:** Routes to backend with fewest active connections. Great for varying request durations (e.g., file uploads vs metadata lookups).
3. **IP Hash (Consistent Hashing):** Deterministically maps a client to a backend. Good for session affinity without a shared store, but beware of uneven load (e.g., corporate NATs).

### Sticky Sessions vs. Shared State
- **Sticky Sessions:** Pinning a user to a server instance. **Avoid this** unless necessary (like WebSockets) as it causes uneven load and failure amplification (server dies = sessions die).
- **Alternative:** Use a shared session store (Redis) or client-side tokens (JWT) to keep the compute tier truly stateless.

### Auto-Scaling
- **Reactive:** Triggers on metrics (CPU, latency). Has inherent lag.
- **Predictive:** Scales ahead of known spikes (e.g., morning traffic).
- *Rule of thumb:* Scale-out should be fast/aggressive; scale-in should be slow/conservative to avoid flapping.

---

## Case Study: Dropbox (File Sync)
- **Requirements:** Sync files across devices, offline editing, very large files, no data loss.
- **Estimations:** 2.5 Exabytes storage. 100M DAU. 100k events/sec peak. Metadata ops (300k-1M/sec) are the true bottleneck, not block storage.
- **Architecture:** 
  - Client Sync Agent (chunks files, delta sync).
  - API Gateway (Stateless, L7 LB using *least-connections*).
  - Metadata Service (Sharded relational DB for file tree).
  - Block Storage (Content-addressed, immutable 4MB chunks, erasure-coded).
  - Notification Service (Pub/Sub for telling devices to fetch changes).
- **Aha! Insights:**
  - **Delta Sync / Block-level sync:** Don't upload a whole 5GB file if 1MB changed. Chunk it, hash it, and only upload the changed chunks.
  - **Optimistic Concurrency:** Detect conflicts at commit time (server returns 409) rather than trying to globally lock a file (which breaks offline editing).
  - **Deduplication:** Content-addressed immutable blocks mean that if two users upload the same file, the bytes are only stored once on the backend.
