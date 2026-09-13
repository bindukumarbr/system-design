# Fast-Track: Module 21 - URL Shortener Build
**Core Concept:** Snowflake IDs -> Base62. Cache-aside. Partial Indexes.
**Case Study:** URL Shortener
- **Key Insight:** Use HTTP 302 (Found) instead of 301 (Moved Permanently) because 301 gets cached by browsers and skips analytics tracking and makes target URL updates impossible.
- **Takeaway:** Use Snowflake IDs to avoid database sequence bottlenecks, and partial indexes (WHERE is_active=TRUE) to keep indexes tiny.
