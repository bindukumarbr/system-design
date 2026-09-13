# NEW Case Study: Web Crawler (Google Search)
- **Requirements:** Crawl billions of pages, don't DDOS targets, prevent infinite loops.
- **Architecture:** URL Frontier (Kafka) + BFS Workers + DNS Cache.
- **Key Insight:** Ensure politeness by grouping the URL frontier queues by domain, processing sequentially with delays. Prevent infinite loops by checking URLs against a Bloom Filter before adding them to the frontier.
