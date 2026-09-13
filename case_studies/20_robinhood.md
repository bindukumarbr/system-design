# Case Study: Robinhood (Serverless Trading)
- **Requirements:** Handle massive, sudden traffic spikes at 9:30 AM market open.
- **Architecture:** AWS Lambda with Provisioned Concurrency.
- **Key Insight:** Lambda cold starts (init time) would destroy trading latency at 9:30 AM. Provisioned Concurrency keeps instances warm ahead of time. Connect to RDS using RDS Proxy to prevent connection exhaustion.
