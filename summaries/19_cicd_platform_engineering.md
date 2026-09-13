# Fast-Track: Module 19 - CI/CD & Platform Engineering
**Core Concept:** Canary/Blue-Green Deployments, Feature Flags, Internal Developer Platforms (Backstage).
**Case Study:** Uber Deployments
- **Key Insight:** Feature flags decouple "deployment" (code to servers) from "release" (enabling feature for users). Use automated canary analysis to auto-rollback on metric regressions.
- **Takeaway:** In-process feature flag evaluation (LaunchDarkly cache) is required for high-scale paths.
