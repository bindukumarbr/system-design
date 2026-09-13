# Case Study: YouTube (Top-K Trending)
- **Requirements:** Find the top 100 most viewed videos out of billions in a sliding window.
- **Architecture:** KEDA autoscaling + Kubernetes StatefulSets.
- **Key Insight:** Memory is the bottleneck for counting billions of unique keys. Use Count-Min Sketch (probabilistic data structure) for heavy hitters to massively reduce RAM footprint in exchange for a small, acceptable margin of error.
