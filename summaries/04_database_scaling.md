# Fast-Track: Module 04 - Database Scaling
**Core Concept:** Read Replicas, Sharding, Partitioning. Connection Pooling (PgBouncer). DynamoDB GSI.
**Case Study:** Uber Eats
- **Key Insight:** Shard by estaurant_id to localize queries. Mitigate hot partitions (e.g., popular restaurants) using composite keys (appending a random suffix).
- **Takeaway:** Understand how your sharding key affects data distribution and hot spots.
