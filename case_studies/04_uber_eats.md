# Case Study: Uber Eats (Restaurant Sharding)
- **Requirements:** Search nearby restaurants, place orders, track delivery.
- **Architecture:** Geohashing for search, heavily sharded database for orders.
- **Data Model:** Sharded by estaurant_id as operations are highly localized to the restaurant.
- **Key Insight:** Hot partitions (e.g., McDonald's at lunch) are mitigated by using a composite partition key (e.g., estaurant_id + random_suffix(1..10)) to spread the write load across multiple nodes.
