# Case Study: Real-Time Chat (Global Production)
- **Requirements:** Multi-region, survive node failures instantly.
- **Architecture:** Active-Active Multi-Region with GeoDNS.
- **Key Insight:** "Online Presence" cannot rely on a sticky session or local node memory. It must be a globally replicated, TTL-expiring fact in a fast datastore (like Redis or Cassandra). When a node crashes, the TTL expires and the user correctly appears offline globally.
