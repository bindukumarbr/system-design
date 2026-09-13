# Fast-Track: Module 01 - System Design Thinking
**Core Concept:** The PEDALS Framework (Problem, Estimate, Design, APIs, Latency/Availability, Scale). CAP Theorem (Consistency, Availability, Partition Tolerance).
**Case Study:** WhatsApp
- **Key Insight:** Separate stateless REST (for accounts/metadata) from stateful connections (WebSockets for real-time messaging).
- **Takeaway:** Always clarify requirements first, estimate load, define API contracts, and then draw the architecture.
