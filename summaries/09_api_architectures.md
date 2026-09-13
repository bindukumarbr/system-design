# Module 09: API Architectures - REST, GraphQL & gRPC

## Core Concepts

### REST & Richardson Maturity Model
- **REST** models nouns (resources), uses standard HTTP verbs (GET, POST), and is **stateless**.
- **Level 0:** Swamp of POX (one endpoint, POST everything).
- **Level 1:** Resources (multiple URIs, still mostly POST).
- **Level 2:** HTTP Verbs (uses GET/PUT/DELETE properly). *This is what 99% of APIs actually are.*
- **Level 3:** HATEOAS (Hypermedia controls, responses include links to next actions). Rare in practice.
- **Versioning:** URI path (`/v1/users`) is standard but un-pure. Headers (`Accept-Version`) are pure but hard to cache/test.

### GraphQL
- Single endpoint (`POST /graphql`). Client asks for exact fields needed.
- Solves **over-fetching** (getting a huge JSON when you only need a name) and **under-fetching** (making 5 API calls to get nested data).
- **The N+1 Problem:** Naively fetching a list of 50 posts and their authors will result in 1 query for posts + 50 queries for authors.
- **Solution:** `DataLoader` pattern (batches all 50 author IDs into a single `WHERE id IN (...)` query and caches them per-request).

### gRPC & Protocol Buffers
- Google's RPC framework over HTTP/2. Uses binary **Protobufs** instead of JSON (compact, fast).
- Generates strongly-typed SDK stubs for clients.
- **Streaming Modes:**
  1. Unary (1 request, 1 response).
  2. Server Streaming.
  3. Client Streaming.
  4. Bidirectional Streaming (Perfect for chat applications).

### API Gateway
- Single ingress point. Handles: Routing, Auth (validating JWTs), Rate Limiting, Protocol Translation (e.g., REST to gRPC).

---

## Case Study: WhatsApp-style Messaging Platform
- **Requirements:** 500M DAU, 1:1 and group chat, delivery receipts (single/double ticks).
- **Estimations:** 20B messages/day (230k/sec to 1M/sec peak). 200M concurrent persistent connections.
- **Architecture (Two Planes):**
  - **REST / Account Plane:** Standard HTTP Gateway -> Account Service -> DB. Handles profile pics, status, etc. (Stateless, cacheable, low traffic).
  - **gRPC / Messaging Plane:** Bidirectional streaming. Millions of long-lived connections.
    - **Message Router:** Uses a Directory Service (Redis/KV) to find which specific server currently holds a user's persistent socket, and routes the message there.
    - **Message Store:** Cassandra/NoSQL partitioned by `conversation_id`.
- **Aha! Insights:**
  - **Why gRPC Bidirectional Streaming?** HTTP/REST is terrible for chat because you can't push messages from the server, and polling 200M clients destroys servers. You need a persistent binary connection.
  - Do NOT put long-lived chat sockets behind a generic HTTP API Gateway. You need a specialized L4/L7 load balancer aware of persistent sockets.
