# Case Study: Rate Limiter Design
- **Requirements:** Throttle excessive API requests per user, highly available, low latency.
- **Architecture:** Redis-based counting at the API Gateway level.
- **Key Insight:** Use Token Bucket for general API throttling (allows bursts). Use Sliding Window Log for strict accuracy (e.g., financial transactions). Execute in Redis using Lua scripts for atomicity.
