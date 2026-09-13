# Fast-Track: Module 23 - Real-Time Chat Build
**Core Concept:** WebSockets, Kafka + Redis Pub/Sub, Presence.
**Case Study:** Chat System
- **Key Insight:** Kafka provides durable, replayable fan-out. Redis Pub/Sub provides ephemeral routing to the specific gateway instance holding the live WebSocket.
- **Takeaway:** Watermark read receipts (store last read ID per user) instead of a row per message read.
