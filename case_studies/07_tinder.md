# Case Study: Tinder (Ordered Matching)
- **Requirements:** Process millions of swipes, notify on match instantly, no lost matches.
- **Architecture:** Event-driven via Kafka.
- **Data Model:** Redis for active user caching, Postgres/NoSQL for match ledger.
- **Key Insight:** Partition Kafka topics by user_id so all swipes for a specific user are processed sequentially by the same consumer, ensuring chronological integrity and preventing race conditions in match logic.
