# Case Study: Reddit (Cache Stampede Prevention)
- **Requirements:** Serve hot posts to millions of concurrent users instantly.
- **Architecture:** Heavy read-through and cache-aside caching (Redis/Memcached).
- **Data Model:** Materialized views of top posts.
- **Key Insight:** When a highly-viewed cache key expires, thousands of threads will hit the DB simultaneously (Thundering Herd). Prevent this via Probabilistic Early Expiration (PER) or locking (only one thread recomputes, others read stale data).
