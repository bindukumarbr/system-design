# Case Study: Yelp (Real-time Reviews)
- **Requirements:** Stream new reviews to business dashboards live without page reload.
- **Architecture:** Server-Sent Events (SSE).
- **Key Insight:** Since the communication is strictly unidirectional (Server pushes new reviews -> Client), SSE is lighter, simpler, and more firewall-friendly than a bidirectional WebSocket.
