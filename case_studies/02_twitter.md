# Case Study: Twitter/X (News Feed)
- **Requirements:** High read volume, fast feed loading, eventual consistency acceptable.
- **Architecture:** Fan-out-on-write vs Fan-out-on-read hybrid model.
- **Data Model:** Redis sorted sets per user for feeds.
- **Key Insight:** Push feed updates (fan-out-on-write) for normal users. Pull updates (fan-out-on-read) at query time for celebrities (e.g., Justin Bieber) to prevent massive write amplification.
