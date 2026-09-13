# Module 11: Microservices Fundamentals

## Core Concepts

### Monolith vs Microservices
- **Monolith-First Strategy:** Start with a modular monolith. Only split into microservices when team friction, deployment blocking, or wildly different scaling profiles force you to.
- Microservices trade **in-process complexity** for **network complexity** (distributed tracing, eventual consistency, partial failures).

### Domain-Driven Design (DDD) Basics
- **Bounded Context:** A clear boundary around a specific business capability (e.g., Billing, Shipping, Auth). In microservices, One Bounded Context = One Service.
- Don't build "technical" microservices (e.g., a "Database Service" or "Validation Service"). Build "business" microservices.

### Data Ownership (No Shared Databases)
- **Golden Rule:** A microservice must exclusively own its data. No other service is allowed to query its tables directly; they MUST use its API.
- **Why?** If Service A and Service B share a DB, and Service A changes a schema, Service B breaks. They are no longer independently deployable. This is a "Distributed Monolith" (the worst of both worlds).
- **Consequences:** You can't do SQL `JOIN`s across microservices. You must use API composition or a CQRS Read Model.

### Synchronous vs Asynchronous Communication
- **Sync (REST/gRPC):** Use when the caller needs an immediate answer (e.g., Auth check, fetching a profile). Risk: Cascading failures.
- **Async (Kafka/RabbitMQ):** Use when things happen "as a result of" an event (e.g., "OrderPlaced" triggers shipping, notifications, and analytics). Good for decoupling and load-leveling.

---

## Case Study: Strava-style Activity Platform
- **Requirements:** Upload GPS tracks, compute distance/elevation, match against Segments (popular routes), update leaderboards, fan-out to social feed.
- **Estimations:** 160 uploads/sec (peak). Feed is read-heavy (100:1 read-to-write ratio). Leaderboard writes are highly skewed to popular routes.
- **Architecture:**
  - **Activity Ingestion Service (Sync):** Accepts GPS file, saves to S3, writes `status: processing` to its own Postgres DB. Emits `ActivityUploaded` event to Kafka. Returns `202 Accepted` immediately to unblock the mobile app.
  - **GPS Worker (Async):** Consumes event, parses GPS, matches segments. Emits `SegmentEffortRecorded`.
  - **Leaderboard Service (Async Consumer):** Consumes `SegmentEffortRecorded`. Saves to Cassandra (Wide-Column store, partitioned by Segment ID for massive write/read scale).
  - **Social Feed Service:** Denormalized document store. Consumes events to build feeds.
- **Aha! Insights:**
  - The Ingestion API handles heavy I/O but returns immediately (202 Accepted). This hides the slow, compute-heavy geospatial matching from the user.
  - **Re-derive from Canonical Storage:** When the Leaderboard service consumes a Kafka event, it doesn't blindly trust it. It queries the canonical effort store and *re-derives* the rank. This makes the system tolerant to out-of-order or duplicate Kafka deliveries.
