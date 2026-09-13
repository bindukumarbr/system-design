# Fast-Track: Module 10 - Real-Time & Async APIs
**Core Concept:** WebSockets (Bidirectional), SSE (Unidirectional Server->Client), Long Polling.
**Case Study:** Yelp Real-time Reviews
- **Key Insight:** Use SSE for live streaming of reviews (client only receives). Use WebSockets for chat (bidirectional).
- **Takeaway:** Don't use WebSockets if communication is one-way; SSE is lighter and runs over standard HTTP.
