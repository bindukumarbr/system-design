# Fast-Track: Module 02 - Scalability Fundamentals
**Core Concept:** Horizontal vs. Vertical Scaling. Sharding (range, hash). Consistent Hashing. Load Balancers (Round Robin, Least Connections, IP Hash).
**Case Study:** Twitter/X
- **Key Insight:** Fan-out-on-write (push to follower timelines) for normal users. Fan-out-on-read (pull at read time) for celebrities to avoid massive write amplification.
- **Takeaway:** Use Redis sorted sets for timelines. Match your fan-out strategy to user behavior.
