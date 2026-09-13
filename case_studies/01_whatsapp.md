# Case Study: WhatsApp (Messaging System)
- **Requirements:** 10B msgs/day, low latency, high availability, offline support.
- **Architecture:** 2-tier. Stateless REST tier for account/metadata (Postgres). Stateful WebSocket tier for real-time messaging.
- **Data Model:** NoSQL (e.g. Cassandra or HBase) for messages to handle high append rate.
- **Key Insight:** Separate the stateless HTTP operations from the stateful TCP/WebSocket connection holding.
