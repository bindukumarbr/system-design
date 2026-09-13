# Module 10: Real-Time & Async API Patterns

## Core Concepts

### WebSockets vs SSE vs Long Polling
- **WebSockets:** Full-duplex (bidirectional) persistent TCP connection. Use when client needs to push data to server as often as it receives it (e.g., chat, collaborative editing, multiplayer games). *Hard to scale because load balancers need sticky sessions and servers hold millions of open sockets.*
- **Server-Sent Events (SSE):** Unidirectional (server -> client) persistent HTTP connection. Uses standard HTTP GET. Built-in reconnection and `Last-Event-ID` tracking. Perfect for one-way feeds (e.g., live sports scores, LLM typing streams, notifications). *Much easier to scale than WebSockets.*
- **Long Polling:** Client sends HTTP request, server holds it open until data is ready, then returns. Client immediately requests again. Good fallback for restrictive corporate firewalls where WebSockets are blocked.

### Webhooks
- Outbound HTTP POST from Provider to Consumer (e.g., Stripe telling you a payment succeeded).
- **Retry Strategy:** Must use exponential backoff + jitter. 
- **Security:** Use HMAC-SHA256 signatures to prove the payload came from the provider, not an attacker.
- **Idempotency:** Webhooks guarantee *at-least-once* delivery. The consumer must deduplicate using an Event ID.

### The BFF (Backend for Frontend) Pattern
- Instead of one generic API for all clients, create one thin API specifically for Web, one for Mobile, one for Partners.
- **Why?** Mobile wants heavily compressed, aggregated data (minimize round trips on 4G). Web can handle chatty REST calls and wants rich data. A BFF tailors the API to the UI.

---

## Case Study: Yelp-style Local Business Platform
- **Requirements:** Search local businesses, view details, post reviews. Real-time updates for wait times and new reviews. Webhook API for reservation partners.
- **Estimations:** Search is heavily read-dominant. 6,000 QPS for search, 4,000 QPS for details. 400,000 concurrent live connections for wait-time updates.
- **Architecture:**
  - **Search:** Denormalized Elasticsearch cluster (eventually consistent).
  - **Realtime Gateway:** An SSE server that holds long-lived connections and subscribes to a Redis/Kafka pub-sub topic. When a wait time updates, it pushes to only the users viewing that business.
  - **Webhook Outbox:** When a reservation is booked, the domain service writes an `outbox` row. A separate Webhook Delivery Service polls the outbox, signs the payload, and sends the POST request, handling retries independently of the domain service.
- **Aha! Insights:**
  - **Why SSE over WebSockets?** The client never needs to push wait-time updates *to* the server. The data flow is strictly server->client. SSE is vastly simpler to operate for 400k concurrent connections.
