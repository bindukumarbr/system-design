# Case Study: Online Auction (eBay)
- **Requirements:** Strict linearizability (no lost bids), high availability.
- **Architecture:** Active-passive per auction, or Single-Writer-Per-Partition.
- **Key Insight:** Route all bids for uction_id=123 to exactly one worker node to guarantee sequential processing (linearizability) without distributed locking overhead. Rely on idempotency keys to handle client retries safely.
