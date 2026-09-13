# Case Study: URL Shortener (Productionizing)
- **Requirements:** Cache massive read traffic, auto-scale.
- **Architecture:** Redis cache-aside (allkeys-lru) + L7 Load Balancing.
- **Key Insight:** Use HTTP 302 (Found) instead of HTTP 301 (Moved Permanently). A 301 caches the redirect in the user's browser forever, bypassing your analytics server and preventing you from ever updating the link destination.
