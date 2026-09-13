# Fast-Track: Module 05 - Caching Fundamentals
**Core Concept:** Cache-aside, Read-through, Write-through, Write-back. Eviction (LRU, LFU). TTL. Cache Stampede.
**Case Study:** Reddit
- **Key Insight:** Prevent cache stampedes (thundering herd) on hot posts by using probabilistic early expiration (recomputing cache just before it expires) or locking.
- **Takeaway:** Cache hit ratio is your most important metric before scaling the DB.
