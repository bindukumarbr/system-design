# NEW Case Study: Uber / Lyft (Ride-Sharing)
- **Requirements:** Match riders to nearest drivers, track driver location in real-time.
- **Architecture:** Geospatial Index + WebSockets.
- **Key Insight:** You cannot query a relational DB for "nearest coordinates" efficiently. Use Geohashing or QuadTrees to map 2D coordinates into 1D strings, allowing fast prefix-matching searches for nearby drivers in Redis or a specialized spatial DB.
