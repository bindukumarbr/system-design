# Case Study: Uber (CI/CD Platform)
- **Requirements:** Safely deploy 4,000 microservices multiple times a day.
- **Architecture:** Automated Canary Analysis + Feature Flags.
- **Key Insight:** Feature flags (LaunchDarkly) decouple code deployment (infrastructure risk) from feature release (business risk). Evaluate flags in-process using an SDK cache to avoid network latency on every if(flag.enabled) check.
