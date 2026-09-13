# Module 23: Real-Time Chat System (Full Build)

## Core Concepts

### Connection Management
- **WebSocket Gateways:** Stateless tier that holds thousands of long-lived TCP connections (e.g., 50k per Node.js/Go instance). 
- **Presence Store (Redis):** "Who is online and which Gateway holds their socket?" Store heartbeats with a TTL in Redis. If a client drops, the TTL expires. No graceful disconnect required.

### Fan-Out Architecture
- **Synchronous Write:** Client sends message -> Gateway -> Chat Service -> Persist to DB (Ack back to client).
- **Asynchronous Publish:** Chat Service publishes to Kafka (partitioned by conversation_id).
- **Fan-Out Workers:** Read Kafka -> lookup online members -> Publish to Redis Pub/Sub channels (keyed by Gateway ID).
- **Final Hop:** Gateway reads Redis Pub/Sub -> pushes down the active WebSocket.

### Storage & Data Model
- **Wide-Column Store (Cassandra/DynamoDB):** Needed for massive write volume. Partition by `conversation_id`, Cluster by `message_id DESC`. 
- **Read Watermarks:** Do not store a "read" row for every user for every message. Store a single `last_read_message_id` per user per conversation.

---

## Case Study: Slack/Discord-style Chat
- **Requirements:** 50M DAU, 5M concurrent WebSockets. 10,000 members per channel.
- **Estimations:** 150,000 msgs/sec at peak. 1.2M deliveries/sec after fan-out.
- **Architecture:** 
  - WebSockets terminate at Gateway instances.
  - Kafka guarantees durability and ordered fan-out.
  - Redis Pub/Sub handles fast cross-instance routing.
- **Aha! Insights:**
  - **Kafka AND Redis:** Why both? Kafka is durable and handles massive parallel fan-out gracefully (if a worker dies, it replays). Redis Pub/Sub is ephemeral but handles the final sub-millisecond hop to the exact Gateway instance holding the live socket. You need both.
  - **Typing Indicators:** Ephemeral. Skip Kafka. Push directly over Redis Pub/Sub and rely on client-side auto-expiry (e.g. timeout after 5s).
