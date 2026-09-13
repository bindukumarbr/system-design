# Case Study: Real-Time Chat (Build)
- **Requirements:** 1-on-1 and Group chat, read receipts, online status.
- **Architecture:** Stateless Gateways + Message Bus.
- **Key Insight:** Kafka is used for durable, replayable fan-out. Redis Pub/Sub is used simultaneously for fast, ephemeral routing to figure out which exact gateway instance holds the recipient's live WebSocket.
