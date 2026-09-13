# Case Study: Netflix (Multi-Tier CDN)
- **Requirements:** Stream high-bandwidth video globally with low buffering.
- **Architecture:** Open Connect Appliances (custom CDNs) embedded directly in ISP networks.
- **Data Model:** Video chunks pre-positioned based on predictive algorithms.
- **Key Insight:** Do not rely on pull-through caching for multi-gigabyte files. Push popular content to edge nodes proactively at off-peak hours (nighttime) to save backbone bandwidth during peak hours.
