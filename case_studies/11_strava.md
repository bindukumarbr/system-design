# Case Study: Strava (Microservice Bounded Contexts)
- **Requirements:** Track workouts, compute analytics, show social feeds.
- **Architecture:** Domain-Driven Design (Activity context, Social context, Analytics context).
- **Data Model:** Service-per-database pattern.
- **Key Insight:** The Activity service owns the Postgres DB. It publishes an ActivityCreated event to an Event Bus (Kafka), which the Social service consumes to fan out to followers' feeds. Services never share a DB.
