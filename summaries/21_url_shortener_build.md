# Module 21: URL Shortener – Backend & Infrastructure (Full Build)

## Core Concepts

### ID Generation Strategies
- **Auto-increment + Base62:** Simple. Use DB sequence, convert to Base62. Bottleneck at DB, predictable IDs.
- **Hash-based (MD5/SHA-256):** Hash long URL, take first 7 chars. Stateless, deterministic. Needs DB roundtrip to handle collisions.
- **Snowflake IDs (Best Choice):** 64-bit integer. 41-bit timestamp, 10-bit worker ID, 12-bit sequence. Generated entirely in-memory, no central DB lock, strictly k-sortable. Convert to Base62 for short URL.

### Database Design
- **Read/Write Skew:** Redirects (Reads) dominate Creates (Writes) by 100:1.
- **Schema:** 
  - `id` (Snowflake BIGINT PK)
  - `short_code` (VARCHAR)
  - `long_url` (TEXT with length check)
- **Indexing:** A Partial Index `ON urls (short_code) WHERE is_active = TRUE` is the most critical index for fast path lookups avoiding dead rows.

### Caching Architecture
- **Cache-Aside Pattern:** Check Redis first. If miss, read Postgres, write to Redis. Keeps Redis optional for correctness.
- **TTL & Eviction:** 24h TTL. Eviction: `allkeys-lru`. Long tail of old links naturally expires out of memory. 
- **Click Analytics:** NEVER update click count synchronously on the hot redirect path. Fire an async event (e.g., SQS) for batch processing.

---

## Case Study: TinyURL/Bit.ly Clone
- **Requirements:** 100M links/month. <10ms p99 latency for redirects.
- **Estimations:** Writes: ~400/sec peak. Reads (redirects): ~38,000/sec peak.
- **Architecture:** 
  - **API:** ECS Fargate (stateless Go/Node containers).
  - **Cache:** ElastiCache Redis.
  - **DB:** RDS PostgreSQL Multi-AZ.
- **Aha! Insights:**
  - **HTTP 302 vs 301:** 301 (Moved Permanently) is cached by browsers aggressively. If you need Click Analytics, you MUST use 302 (Found). With 301, the browser bypasses your server on repeat visits.
  - **Avoid Write Bottlenecks:** Offload click increments. An `UPDATE urls SET clicks = clicks + 1` on every redirect would destroy the Postgres DB under heavy read load. Use a decoupled SQS queue.
