# Fast-Track: Module 11 - Microservices Fundamentals
**Core Concept:** Domain-Driven Design (DDD), Service Discovery, API Gateway, Service-per-Database.
**Case Study:** Strava
- **Key Insight:** Decompose by bounded contexts (Activity, Social, Analytics). The Activity service owns its Postgres shard. Event bus updates Social feeds.
- **Takeaway:** Microservices must own their data independently to prevent coupling.
