# Fast-Track: Module 22 - URL Shortener Productionizing
**Core Concept:** Auto-scaling, Rate Limiting (Token Bucket), CDN caching gotchas.
**Case Study:** URL Shortener
- **Key Insight:** Target-tracking auto-scaling based on requests/instance. Cache hit ratio dropping is the leading indicator of a DB overload.
- **Takeaway:** Protect create endpoints with Token Bucket rate limiting in Redis.
