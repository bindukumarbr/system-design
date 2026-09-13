# Fast-Track: Module 07 - Messaging Systems
**Core Concept:** AMQP vs Kafka. Guarantees (At-most/At-least/Exactly-once). Topics, Partitions, Consumer Groups. DLQ (Dead Letter Queue).
**Case Study:** Tinder
- **Key Insight:** Kafka ensures ordered processing of swipe events. Partition by user_id to parallelize without losing order for a specific user.
- **Takeaway:** Use message queues to decouple services, absorb spikes, and ensure durability of async tasks.
