# Fast-Track: Module 24 - Scaling Chat to Production
**Core Concept:** Multi-region, Global Presence, Chaos Testing.
**Case Study:** Chat System
- **Key Insight:** Sticky sessions break in multi-region. Global presence must be a replicated, TTL-based fact store (heartbeats) rather than implicit node-local state.
- **Takeaway:** Use cross-region Kafka mirroring (MirrorMaker 2) only when conversations span multiple regions, keeping local traffic local.
