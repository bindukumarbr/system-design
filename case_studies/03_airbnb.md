# Case Study: Airbnb (Booking System)
- **Requirements:** No double-booking, transactional correctness, heavy reads for searching.
- **Architecture:** Monolithic or domain-driven microservices with a strong ACID database at the core.
- **Data Model:** PostgreSQL.
- **Key Insight:** Rely on SQL transaction isolation levels (e.g., Repeatable Read / Serializable) and row locks (SELECT FOR UPDATE) to prevent concurrent double-booking of a single property.
