# Fast-Track: Module 08 - Event-Driven Architecture & CQRS
**Core Concept:** Event Sourcing, CQRS (Command Query Responsibility Segregation), Saga Pattern (Choreography/Orchestration), Outbox Pattern, CDC (Change Data Capture).
**Case Study:** Online Judge (LeetCode)
- **Key Insight:** Event-sourced submission lifecycle. CQRS separates heavy read models (leaderboards) from write models (code submissions). Saga coordinates test runners.
- **Takeaway:** Use Outbox Pattern + CDC (Debezium) to safely publish events from database commits.
