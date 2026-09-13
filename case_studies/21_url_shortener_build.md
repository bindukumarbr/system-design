# Case Study: URL Shortener (Build)
- **Requirements:** Generate short aliases, redirect fast, track clicks.
- **Architecture:** Base62 Encoding + Distributed ID Generator.
- **Key Insight:** An auto-increment DB sequence is a bottleneck. Use Snowflake IDs (Time + Worker ID + Sequence) to generate unique 64-bit integers locally, then encode them using Base62 to get the short URL.
