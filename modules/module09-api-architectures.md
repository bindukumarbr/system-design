# Module 9: API Architectures - REST, GraphQL & gRPC

## Core Concepts

### REST Resource Modeling and Statelessness

REST (Representational State Transfer) models a system as a set of **resources** (nouns — `users`, `orders`, `messages`), each addressed by a URI, manipulated through a small uniform set of verbs (GET, POST, PUT, PATCH, DELETE) and represented in a transferable format (JSON is the de facto standard today). Good resource modeling:

- Uses plural nouns for collections (`/users/123/conversations`), never verbs (`/getUserConversations`).
- Models relationships via nested or linked resources rather than RPC-style endpoints.
- Uses HTTP status codes correctly (`201 Created` + `Location` header on POST, `404` vs `410`, `409` for conflicts, `429` for rate limiting).
- Uses idempotency correctly: GET/PUT/DELETE are idempotent by contract, POST is not — this matters for retry safety on flaky networks.

**Statelessness** is the constraint that each request contains all the information the server needs to process it (auth token, needed context) — the server holds no per-client session between requests. This is what lets REST APIs scale horizontally behind a load balancer without sticky sessions or shared session stores, at the cost of pushing state to the client or to a shared cache/DB (e.g., a JWT or an opaque token validated against Redis).

### Richardson Maturity Model

Leonard Richardson's model gives four levels of "how RESTful" an API actually is:

- **Level 0 — The Swamp of POX.** A single URI endpoint, single HTTP verb (usually POST), and the payload itself (often XML or JSON-RPC style) encodes the actual operation, e.g. `POST /api` with body `{"method":"getUser","id":123}`. HTTP is used purely as a transport tunnel; nothing about the URI or verb is meaningful. Classic SOAP/XML-RPC endpoints live here.
- **Level 1 — Resources.** The API introduces individual resource URIs (`/users/123`, `/orders/456`) instead of one god-endpoint, but still typically funnels everything through POST for every operation. You get addressability but not a uniform interface.
- **Level 2 — HTTP Verbs.** The API now uses HTTP methods semantically (GET for reads, POST for creates, PUT/PATCH for updates, DELETE for removal) and HTTP status codes to communicate outcome. This is where the overwhelming majority of production "REST APIs" (Stripe, GitHub, Twilio) actually sit — it captures nearly all of REST's practical benefit: cacheability, correct semantics, tooling support.
- **Level 3 — Hypermedia Controls (HATEOAS).** Responses embed links describing available next actions (e.g., an order response includes a `cancel` link only if cancellation is currently legal), so clients discover valid transitions at runtime instead of hard-coding them. This is the "purist" REST Fielding described in his dissertation, but it's rare in practice — client/server coupling to a fixed contract is usually considered an acceptable trade for simplicity, and most API consumers (mobile apps, generated SDKs) don't consume hypermedia dynamically anyway.

Interview framing: know that "REST" as commonly practiced is Level 2, and be able to explain precisely what Level 3 adds and why it's rarely adopted (client complexity, tooling/codegen mismatch, marginal benefit unless the state machine is large and evolving).

### API Versioning Strategies

| Strategy | Mechanism | Pros | Cons |
|---|---|---|---|
| URI path (`/v1/users`) | Version baked into the URL | Simple, cache-friendly (different URL = different cache key), visible in logs/browser, easy to route at gateway/LB layer | "Breaks" REST purity (a resource's identity arguably shouldn't change with version); tempts duplicating whole route trees |
| Custom header (`X-API-Version: 2`) or `Accept-Version` | Client sends a header, server dispatches internally | Keeps URI stable/canonical; multiple versions can share caching keyed on URI+header | Less visible/discoverable, harder to test with a browser, easy to forget in client code, proxies/CDNs may not vary cache on custom headers by default |
| Content negotiation (media-type versioning, e.g. `Accept: application/vnd.myapi.v2+json`) | Version embedded in the MIME type via the standard `Accept` header | Most "correct" REST-theoretically (a resource is one URI, representation varies); plays well with HTTP caching semantics | Highest client complexity; poor tooling/browser support; awkward with codegen |
| No versioning / additive evolution | Only add optional fields, never remove/rename; deprecate via sunset headers | No version proliferation | Requires strict discipline; eventually still needs a breaking-change path |

Real-world practice: Stripe and GitHub use a **date-based version header/URI** combined with strict additive-change rules, converting the "n breaking versions to maintain" problem into "translate old-dated requests into current internal model." For interviews, the key trade-off to articulate is: URI versioning wins on operational simplicity and routability at the gateway (a load balancer can route `/v2/*` to a different fleet); header/content-negotiation versioning wins on URI purity and cache-key economy but costs discoverability.

### GraphQL — Schema, Resolvers, Queries, Mutations, Subscriptions

GraphQL exposes a single endpoint (typically `POST /graphql`) governed by a strongly typed **schema** (SDL: `type User { id: ID!, name: String!, posts: [Post!]! }`). The client sends a query describing the exact shape of data it wants; the server executes it against a graph of **resolver** functions — one function per field — that fetch/compute that field's value, often calling into the parent resolver's returned object.

- **Queries** are reads: the client asks for exactly the fields it needs, in one round trip, even across relationships (e.g., user → their last 5 orders → each order's line items) — this eliminates classic REST over-fetching (getting a whole user object when you needed just the name) and under-fetching (needing N follow-up calls for related resources).
- **Mutations** are named write operations (`createOrder(input: ...): Order`) — GraphQL doesn't have HTTP-verb semantics; the "REST verb" is baked into the mutation's name and executed server-side.
- **Subscriptions** provide server-push over a persistent transport (historically WebSocket, more recently `graphql-sse`) for live updates the client has subscribed to (e.g., "notify me when this order's status changes").

**The N+1 resolver problem**: a naive resolver for `posts { author { name } }` over 50 posts calls the "get author by id" resolver once per post — 50 individual DB round-trips (plus the 1 for posts) instead of one. The standard fix is a **batching/caching loader** (the `DataLoader` pattern, popularized by Facebook): within a single GraphQL execution tick, individual `load(id)` calls are collected into a queue and flushed as one batched query (`WHERE id IN (...)`) per unique key, with per-request memoization so the same id is never fetched twice. This is one of the most commonly probed GraphQL interview topics — know DataLoader by name and describe the batch-and-cache mechanism precisely, not just "add caching."

Other structural GraphQL concerns worth naming: schema stitching/federation (Apollo Federation) for splitting one graph across services, persisted queries (to avoid sending large query documents over the wire and to allowlist accepted queries for security), and query complexity/depth limiting (to prevent a malicious deeply-nested query from causing a resolver explosion — GraphQL has no built-in rate limiting the way REST endpoints do per-route).

### gRPC — Protocol Buffers, Service Definitions, Streaming Modes

gRPC is Google's RPC framework built on HTTP/2, using **Protocol Buffers (protobuf)** as the interface definition language and wire format. A `.proto` file defines message types and a service:

```protobuf
syntax = "proto3";
service MessageService {
  rpc SendMessage (SendMessageRequest) returns (SendMessageResponse);
  rpc StreamIncoming (StreamRequest) returns (stream IncomingMessage);
  rpc UploadDeliveryReceipts (stream DeliveryReceipt) returns (Ack);
  rpc Chat (stream ChatMessage) returns (stream ChatMessage);
}
```

Protobuf messages are binary-encoded with field numbers (not names) on the wire, giving compact payloads and forward/backward compatibility rules (new optional fields don't break old clients; never reuse a field number). Code generation produces strongly typed client/server stubs in every major language — this eliminates hand-written serialization code and an entire class of contract-mismatch bugs that REST/JSON APIs get only via extra tooling like OpenAPI codegen.

The **four streaming modes**, precisely:
1. **Unary** — one request, one response (`SendMessage`), the RPC analog of a normal REST call.
2. **Server streaming** — one request, a stream of responses (`StreamIncoming`) — server keeps pushing until it closes the stream. Ideal for "subscribe to a feed of events."
3. **Client streaming** — a stream of requests, one final response (`UploadDeliveryReceipts`) — client sends many messages, server acks once at the end. Good for batch upload / accumulate-then-respond.
4. **Bidirectional streaming** — both sides stream independently over one long-lived HTTP/2 connection (`Chat`) — this is the closest RPC analog to a persistent socket and is what makes gRPC attractive for real-time messaging: a single multiplexed connection carries many concurrent logical streams (HTTP/2 multiplexing) with low per-message framing overhead, keep-alive, and flow control built in.

### API Gateway Responsibilities

An API gateway is the single ingress point that decouples external API contracts from internal service topology. Core responsibilities:

- **Routing** — path/host-based dispatch to the correct backend service or version, often with weighted routing for canary releases.
- **Authentication & authorization** — terminate/validate JWTs or OAuth tokens once at the edge instead of duplicating auth logic in every service; attach a verified identity/claims header downstream.
- **Rate limiting & quota enforcement** — per-client/per-key token buckets or sliding windows, protecting backends from abuse and enabling tiered API plans.
- **Request/response transformation** — protocol translation (e.g., expose REST/JSON externally, gRPC internally — "gRPC-gateway" pattern), header injection, request aggregation (fan-out to multiple services and compose one response), and response shaping.
- Secondary duties commonly bolted on: TLS termination, request logging/tracing (inject trace IDs), circuit breaking, and caching of idempotent GET responses.

## Case Study Solution: WhatsApp-style Messaging Platform

### Problem Statement & Clarifying Requirements

Design a messaging platform (WhatsApp-like) supporting:

**Functional requirements**
- Account creation/login (phone-number based), profile management (display name, avatar, status).
- 1:1 and group messaging (text, media).
- Delivery receipts (sent → delivered → read, the classic single/double/blue-tick model).
- Online/last-seen presence (stretch).
- Multi-device support (stretch, out of deep scope here).

**Non-functional requirements**
- Low end-to-end latency for message delivery (sub-second, ideally ~100–250 ms globally).
- High availability (messaging must survive regional failures).
- At-least-once delivery with client-side dedup (never silently drop a message).
- Massive concurrency: hundreds of millions of simultaneous persistent connections.
- Horizontal scalability of both the connection layer and the account/profile API layer independently.

### Capacity Estimation

Assume 500M daily active users (DAU), each sending ~40 messages/day, in line with WhatsApp's publicly cited ~100B messages/day at 2B+ MAU scale ([ByteByteGo](https://blog.bytebytego.com/p/how-whatsapp-handles-40-billion-messages)):

- Messages/day ≈ 500M × 40 = 20B messages/day → ~230K messages/sec average, with a peak multiplier of 3–5× → ~1M messages/sec peak.
- Concurrent connections: if ~40% of DAU are online at peak → ~200M concurrent persistent connections. WhatsApp's real published figure was ~2–3M connections per physical server using Erlang/BEAM's lightweight process model ([betterengineers.substack.com](https://betterengineers.substack.com/p/how-whatsapp-handled-1-billion-users)) — so 200M connections needs on the order of 100–200 connection-handling nodes purely for socket fan-out, well below what a naive thread-per-connection model would require (which would need low millions of threads and be infeasible).
- Storage: average message ~100 bytes text metadata + media handled separately via blob storage/CDN with a message record pointing at it. 20B messages/day × ~200 bytes (text + envelope) ≈ 4 TB/day of message-log writes before replication factor.
- Account/profile API: much lower QPS — logins, profile fetches, contact sync maybe 50K–100K QPS peak, orders of magnitude below the messaging path, and read-heavy — a classic case for a conventional REST/HTTP stack behind caches.

### High-Level Architecture

```
                         ┌─────────────────────┐
        Mobile/Web  ───► │   API Gateway (HTTPS)│──► Account/Profile Service (REST)
        Clients          │  authn, rate limit,   │        │
        (account ops)    │  routing, TLS term    │        ▼
                         └─────────────────────┘   Users DB (sharded, replicated)
                                                      + Redis cache

        Mobile Clients                     Connection/Session Layer
        (persistent conn) ───TLS/TCP───►  ┌───────────────────────────┐
                                            │ gRPC bidi-stream gateway   │
                                            │ (per-user long-lived conn) │
                                            └───────────┬───────────────┘
                                                         │ presence + routing
                                                         ▼
                                            ┌───────────────────────────┐
                                            │  Message Router / Session  │
                                            │  Directory (which server   │
                                            │  holds each user's socket) │
                                            └───────────┬───────────────┘
                                                         ▼
                                     ┌────────────────────────────────────┐
                                     │  Message Store + Delivery Queue      │
                                     │  (per-conversation log, Kafka-like   │
                                     │   durable queue for offline fan-out) │
                                     └────────────────────────────────────┘
                                                         │
                                          Push Notification Service (APNs/FCM)
                                          for offline delivery wake-up
```

Two clearly separated planes: a conventional **stateless REST API plane** for account/profile CRUD (fits a normal load-balancer + service + DB pattern), and a **stateful persistent-connection plane** for messaging, where each connection-handling node holds an in-memory session table mapping `user_id → socket`, backed by a directory service (e.g., built on a distributed KV store) so any node can look up "which server currently holds Bob's socket" to route a message.

### API Design

**REST — account/profile:**

```
POST /v1/accounts/register
  { "phone": "+14155550123", "otp": "482913" }
  → 201 { "user_id": "u_9f2...", "access_token": "...", "refresh_token": "..." }

GET /v1/users/me/profile
  Authorization: Bearer <token>
  → 200 { "user_id": "u_9f2...", "display_name": "Ada", "avatar_url": "...", "status": "Busy" }

PATCH /v1/users/me/profile
  { "display_name": "Ada L." }
  → 200 { ...updated profile... }

GET /v1/users/me/contacts?cursor=...&limit=100
  → 200 { "contacts": [...], "next_cursor": "..." }
```

**gRPC — messaging path:**

```protobuf
syntax = "proto3";

message Message {
  string message_id = 1;
  string conversation_id = 2;
  string sender_id = 3;
  bytes ciphertext = 4;       // end-to-end encrypted payload
  int64 client_timestamp = 5;
}

enum DeliveryState { SENT = 0; DELIVERED = 1; READ = 2; }

message DeliveryReceipt {
  string message_id = 1;
  string conversation_id = 2;
  DeliveryState state = 3;
  string user_id = 4;
  int64 timestamp = 5;
}

service MessagingService {
  // Bidirectional: client streams outgoing messages + receipts,
  // server streams incoming messages + receipts, over one long-lived connection.
  rpc Connect (stream ClientEnvelope) returns (stream ServerEnvelope);
}

message ClientEnvelope {
  oneof payload {
    Message message = 1;
    DeliveryReceipt receipt = 2;
    Heartbeat heartbeat = 3;
  }
}

message ServerEnvelope {
  oneof payload {
    Message message = 1;
    DeliveryReceipt receipt = 2;
    Ack ack = 3;
  }
}
```

A single `Connect` bidi-stream RPC per device carries all message sends, incoming pushes, and receipt acks multiplexed over one HTTP/2 connection — this collapses what would otherwise be many separate polling/REST calls into one persistent channel.

### Data Model

```
users(user_id PK, phone_number UNIQUE, display_name, avatar_url, status_text, created_at)

devices(device_id PK, user_id FK, push_token, public_key, last_seen_at)

conversations(conversation_id PK, type ENUM('1:1','group'), created_at)

conversation_members(conversation_id FK, user_id FK, joined_at, role, PRIMARY KEY(conversation_id, user_id))

messages(message_id PK, conversation_id FK (partition key), sender_id FK,
         ciphertext BYTES, server_timestamp, client_timestamp)
  -- partitioned/sharded by conversation_id for locality of a chat's history

delivery_status(message_id FK, user_id FK, state ENUM('sent','delivered','read'), updated_at,
                 PRIMARY KEY(message_id, user_id))
  -- one row per (message, recipient) — enables per-recipient read receipts in groups
```

`messages` is the highest-volume table and is sharded by `conversation_id` so a chat's full history is co-located; `delivery_status` is the second-highest-volume table (fan-out per recipient in groups) and is often kept in a separate high-write-throughput store (e.g., a wide-column store or Redis-backed structure) rather than the same relational engine as `users`.

### Deep Dive

**Why REST fits account/profile.** This traffic is classic CRUD over a resource graph (users, contacts, profile fields), read-heavy, cacheable (profile GETs can sit behind a CDN/edge cache with short TTLs and cache invalidation on writes), and consumed by clients that benefit from standard HTTP tooling — browsers, curl, API gateways, OpenAPI-generated SDKs, and infra (LBs, WAFs, CDNs) that already understands HTTP semantics natively. There's no need for a persistent connection or streaming here; the request/response, stateless model is a perfect match and lets this tier scale independently and boringly.

**Why gRPC (persistent binary protocol) beats REST/GraphQL for the messaging path.** Three requirements REST/GraphQL don't serve well: (1) **server push** — a recipient must be notified the instant a message arrives, which requires the server to hold a live channel to the client, not just answer a client-initiated request; (2) **connection efficiency at scale** — HTTP/2 multiplexing plus binary protobuf framing gives far lower per-message overhead and lower latency than repeated HTTP/1.1 JSON requests or GraphQL query parsing per message, which matters at ~1M msg/sec; (3) **low-latency bidirectional flow** — sends, incoming pushes, and delivery receipts all need to flow concurrently over one channel, which is exactly gRPC's bidirectional-streaming mode. GraphQL subscriptions can technically push data, but they add resolver/schema-execution overhead per event and are designed for client-driven, shape-flexible reads, not for a high-throughput, fixed-schema, binary transport — the wrong tool for a hot path processing hundreds of thousands of tiny messages per second. In practice, WhatsApp doesn't use gRPC specifically — it built its own lightweight binary protocol atop TCP/TLS on Erlang/BEAM ([scalewithchintan.com](https://scalewithchintan.com/blog/whatsapp-erlang-architecture-2-billion-users)) — but the architectural reasoning (persistent binary channel, not stateless HTTP request/response) is the same; gRPC is the standard, portable way to get equivalent properties without hand-rolling a wire protocol.

**Where the API gateway sits.** It fronts only the REST/account plane (registration, profile, contact sync, media upload URLs) — auth (validating phone-based session tokens), rate limiting registration/OTP endpoints against abuse, routing to versioned backend services, and TLS termination. The messaging plane deliberately bypasses a traditional HTTP API gateway; instead, a purpose-built **connection gateway/load balancer** (often a custom L4/L7 proxy aware of gRPC/HTTP2 and long-lived-connection load balancing) accepts and load-balances the persistent streams, since a generic HTTP gateway optimized for short request/response cycles is a poor fit for millions of long-lived connections.

**Versioning strategy chosen.** URI-path versioning (`/v1/...`) for the REST account API — it's operationally simplest to route at the gateway/load-balancer layer and is what most public messaging-adjacent APIs (Twilio, Stripe-style) use; for the gRPC messaging protocol, versioning is handled via protobuf's built-in forward/backward compatibility (only add new optional fields, never remove/renumber) plus a `protocol_version` field in the connection handshake, avoiding the need for separate versioned RPC services for every wire change.

### Trade-offs and Alternatives Considered

- **Persistent TCP/gRPC vs long-polling/HTTP:** long-polling is simpler to run through generic HTTP infra but has materially higher latency and connection-churn overhead at this scale — rejected for the primary channel, though a lightweight REST/long-poll fallback is reasonable for very constrained networks.
- **GraphQL for account/profile instead of REST:** viable, and arguably nicer for a rich client that wants to fetch profile+contacts+settings in one round trip — but it adds resolver/N+1 complexity and a less mature caching story for what is fundamentally a small, stable resource set; REST's simplicity wins for this bounded domain, and GraphQL's benefit grows mainly when the read shape is highly variable across many client surfaces (this doc's Module recommends GraphQL when you have many heterogeneous frontends against a large, deeply nested resource graph — not the case here).
- **Single shared connection-and-API gateway:** simpler ops (one ingress), but couples the wildly different scaling/latency profiles of CRUD traffic and persistent-connection traffic — rejected in favor of two independently scaled planes.
- **At-least-once vs exactly-once delivery:** chosen at-least-once with client-side message-id dedup, since exactly-once across a distributed queue + flaky mobile network is prohibitively complex for marginal benefit; clients already need idempotent handling for retries.

### How Real Systems Solve This

WhatsApp famously ran its backend on Erlang/BEAM, whose lightweight process model let a single physical server hold **millions of concurrent connections** (reported ~2–3M per box), a scale that a thread-per-connection design in most other languages could not approach without an equivalent async/event-loop runtime ([betterengineers.substack.com](https://betterengineers.substack.com/p/how-whatsapp-handled-1-billion-users), [scalewithchintan.com](https://scalewithchintan.com/blog/whatsapp-erlang-architecture-2-billion-users)). Their transport is a custom lightweight binary protocol over TCP/TLS rather than off-the-shelf HTTP, prioritizing minimal per-message overhead — the same motivation that leads teams choosing gRPC today to pick bidirectional streaming over JSON/REST for chat-style workloads. Signal similarly separates account/registration (conventional HTTPS/REST-ish APIs) from its message-delivery path, layered on the double-ratchet E2E-encryption protocol, delivered over WebSocket to mobile clients. Facebook Messenger's public engineering material likewise describes fanning out delivery via durable per-user queues so offline recipients still get messages the moment they reconnect — the same "durable delivery queue + push-notification wakeup" pattern used above ([getstream.io architecture overview](https://getstream.io/blog/whatsapp-works/), [ByteByteGo overview](https://blog.bytebytego.com/p/how-whatsapp-handles-40-billion-messages)). The consistent real-world pattern across all three: stateless HTTP/REST for account plane, a custom persistent binary/streaming protocol for the messaging hot path, and a durable queue plus push-notification service bridging the gap for offline recipients.

## Sources

- [How WhatsApp Handles 40 Billion Messages Per Day — ByteByteGo](https://blog.bytebytego.com/p/how-whatsapp-handles-40-billion-messages)
- [How WhatsApp Handled 1 Billion Users with 50 Engineers](https://betterengineers.substack.com/p/how-whatsapp-handled-1-billion-users)
- [How WhatsApp Works - Architecture Deep Dive on 100 Billion Messages — getstream.io](https://getstream.io/blog/whatsapp-works/)
- [WhatsApp Erlang Architecture | Scaling to 2 Billion Users](https://scalewithchintan.com/blog/whatsapp-erlang-architecture-2-billion-users)
- [Richardson Maturity Model — restfulapi.net](https://restfulapi.net/richardson-maturity-model/)
- [Richardson Maturity Model — Wikipedia](https://en.wikipedia.org/wiki/Richardson_Maturity_Model)
- [Know how RESTful your API is — Red Hat Developer](https://developers.redhat.com/blog/2017/09/13/know-how-restful-your-api-is-an-overview-of-the-richardson-maturity-model)
- [Core concepts, architecture and lifecycle — grpc.io](https://grpc.io/docs/what-is-grpc/core-concepts/)
- [Understanding gRPC Streaming: Unary, Server, Client, and Bidirectional RPCs](https://medium.com/@63abhikumar/understanding-grpc-streaming-in-go-unary-server-client-and-bidirectional-rpcs-8e298df07ae8)
