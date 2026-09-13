# Case Study: Online Judge / LeetCode
- **Requirements:** Run untrusted code safely, update leaderboards, track submission history.
- **Architecture:** Event Sourcing & CQRS.
- **Data Model:** Immutable event log (e.g., Kafka) is the source of truth. Read models (Leaderboards) are updated asynchronously.
- **Key Insight:** Use the Outbox Pattern and Change Data Capture (Debezium) to guarantee that when a submission is saved to the DB, the event is guaranteed to be published to Kafka.
