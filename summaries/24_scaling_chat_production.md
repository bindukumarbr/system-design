# Module 24: Scaling Chat Systems for Production (Multi-Region)

## Core Concepts

### Multi-Region Architecture
- **GeoDNS / Anycast:** Route the user to the closest region to minimize WebSocket ping/pong latency.
- **Cross-Region Relay:** If User A (US) talks to User B (EU), local US Kafka mirrors the topic to EU Kafka via MirrorMaker 2. The EU Gateway pushes to User B.

### Multi-Region Presence
- **The Sticky Session Trap:** "Online" cannot mean "connected to my local server". Users have multiple devices across regions.
- **Replicated Presence:** Write heartbeats with TTL to a local Redis. Asynchronously replicate (CDC or Active-Active CRDT) to all other regions. Every region has an eventually consistent global view of "who is online".

### File Uploads (S3 Presigned POST)
- **Proxy Anti-Pattern:** Never pipe a 50MB video through your WebSocket or Chat API servers. It blocks the event loop and wastes bandwidth.
- **Presigned POST:** Client requests upload permission. Server returns an S3 Presigned URL + Policy (limits file size, type). Client uploads directly to S3. S3 triggers an EventBridge/Lambda worker to scan for viruses and publish an "Attachment Ready" message to Kafka.

### Notification Pipeline
- **De-duplication:** Push notifications (APNs/FCM) consume the *exact same* Kafka topic as live chat. If the user is offline (via Presence Store), queue a push. 
- **Debounce:** Wait 5-10 seconds before pushing. Collapse 5 rapid messages into one "5 new messages from Alice" notification.

---

## Case Study: Global Messaging App
- **Requirements:** Users scattered across continents. Near-zero message loss during regional outages.
- **Estimations:** Cross-continent RTT dictates that synchronous cross-region writes will fail latency budgets.
- **Architecture:** 
  - Regional gateways, Kafka, and Redis.
  - Kafka MirrorMaker handles cross-region delivery.
  - S3 Presigned URLs for media.
- **Aha! Insights:**
  - **Failure Ambiguity & TTLs:** If a US Gateway node crashes, how does the EU region know those users are offline? By using TTL-based presence, the heartbeats simply stop replicating. After 30 seconds, the EU region naturally expires the users and marks them offline. No "server died" global broadcast is needed.
  - **Sticky Sessions as Optimization, Not Correctness:** Keep clients stuck to one Gateway for fast TLS and memory pooling. But if that affinity breaks, the TTL presence and Kafka fan-out guarantees they still get their messages.
