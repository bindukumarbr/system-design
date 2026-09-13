# Module 19: CI/CD & Platform Engineering

## Core Concepts

### CI/CD Pipeline Stages
1. **Continuous Integration (CI):** Run on PR. Lint, Unit Tests, Static Code Analysis, Dependency scanning.
2. **Build:** Create Docker image, tag with Git SHA (never `latest`), generate SBOM.
3. **Continuous Deployment (CD):** GitOps tool (ArgoCD) pulls the new manifest and updates the cluster.

### Rollout Strategies
- **Rolling Update:** Replaces pods 1 by 1. Zero downtime, but bad code affects a growing number of users before detection. Slow rollback.
- **Blue-Green:** Spin up 100% new infrastructure (Green). Swap the router. Instant rollback, but costs 2x infrastructure during the deploy.
- **Canary:** Route 1% of traffic to the new version. Automated metric analysis (Error rate, Latency) checks if it's healthy. If yes, 10% -> 50% -> 100%. If no, automatic rollback. Safest, but requires a Service Mesh (Istio) to split traffic.

### Feature Flags
- Decouples *Deployment* (code on server) from *Release* (users seeing the feature).
- Allows you to test in production, gradually roll out a feature, or hit a kill switch without needing to rollback the actual deployment.
- Flags are evaluated in-memory using an SDK (e.g., LaunchDarkly), so checking a flag takes microseconds, not a network call.

### Internal Developer Platforms (IDPs)
- E.g., Backstage (by Spotify). 
- Provides "Golden Paths" so developers can click a button and get a fully scaffolded microservice with CI/CD, monitoring, and security already wired up, without needing to know Kubernetes YAML.

---

## Case Study: Uber-style Deployment Pipeline
- **Requirements:** 4,000 microservices deployed multiple times a day. Safety critical (pricing, dispatch). Must not strand riders.
- **Estimations:** 12,000 deploys a day company-wide.
- **Architecture:** 
  - CI builds image -> ArgoCD initiates deploy -> Argo Rollouts handles Canary traffic splitting -> Prometheus analyzes metrics -> Auto-rollback on error.
- **Aha! Insights:**
  - **Blast Radius Containment:** For a pricing service, Blue-Green is too dangerous (a bug hits 100% of riders instantly). Canary bounds the blast radius mathematically. At 1% traffic, a severe bug only hits 1% of riders for 5 minutes before the automated metrics system detects the error spike and aborts the deploy.
  - **Flag vs Canary:** Canary asks "Is the infrastructure stable? Are there exceptions?" Feature Flags ask "Is this new pricing algorithm making more money?" They solve two different risks.
