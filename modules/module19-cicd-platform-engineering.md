# Module 19: CI/CD, Platform Engineering

## Core Concepts

### 1. CI/CD Pipeline Design (GitHub Actions → Docker → Kubernetes)

A production CI/CD pipeline is a chain of trust: each stage must produce evidence that the next stage can rely on without re-validating it.

**Stage 1 — Commit & static checks (CI).** A pull request triggers a GitHub Actions workflow (`on: pull_request`). Jobs run in parallel: lint, unit tests, type checks, `SAST` (static application security testing), dependency/license scanning (e.g., `trivy`, `snyk`). Each job is a separate Actions job so failures are attributable and jobs can run on a matrix (multiple language/OS versions). Branch protection rules require these checks to pass before merge.

**Stage 2 — Build & package.** On merge to `main` (or a release branch), the workflow builds a container image with Docker, tags it with the immutable git SHA (never `latest`), and pushes it to a registry (ECR/GCR/Artifactory). Multi-stage Dockerfiles keep the runtime image minimal; image signing (`cosign`) and SBOM generation happen here so the artifact is provenance-tracked (SLSA-style).

**Stage 3 — Deploy (CD).** A separate CD system (Argo CD/Spinnaker/Flux, or a GitHub Actions job that calls `kubectl`/Helm) picks up the new image tag. GitOps pattern: CI writes the new tag into a Kubernetes manifest/Helm values file in a config repo; Argo CD reconciles the cluster to match that repo. This separates "what changed" (git history) from "what's running" (cluster state), giving an audit trail and one-command rollback (`git revert`).

**Stage 4 — Progressive delivery.** The new version is exposed to a small slice of traffic (canary) behind a service mesh or ingress controller (Istio, Linkerd, AWS App Mesh, or Argo Rollouts/Flagger), with automated metric analysis gating promotion.

**Environment promotion:** dev → staging → canary → full production, with increasing approval gates (automatic in dev, manual/quorum in prod for safety-critical services).

Key CI/CD principles: pipelines must be **idempotent and repeatable**, artifacts must be **immutable and versioned**, and every production change must be **traceable to a commit and a human or automated approval**.

### 2. Rollout Strategies

**Rolling update (default in Kubernetes Deployments).** Pods are replaced incrementally: `maxSurge` extra pods are created running the new version, then `maxUnavailable` old pods are terminated, repeating until all pods are new. Pros: no extra infrastructure cost, zero-downtime if readiness probes are correct. Cons: both versions run simultaneously for the whole rollout window (must be backward/forward compatible — schema and API contracts), and a bad version affects a growing fraction of traffic before anyone notices; rollback means running the rolling update in reverse, which is not instant.

**Blue-green deployment.** Two full production environments ("blue" = current, "green" = new) run side by side; traffic is cut over atomically (load balancer/DNS/service selector swap) once green is fully validated. Rollback is instant — flip the router back to blue. Cost: 2x infrastructure during the switch, and stateful services (databases, in-flight sessions) need careful handling since the cutover is all-or-nothing per request, not gradual. Best for: services where you want a hard, instantaneous, all-or-nothing switch and can afford double capacity (e.g., low-replica-count, high-value services), and where a canary's gradual exposure is unnecessary or too slow.

**Canary deployment.** A new version is deployed to a small subset of infrastructure/traffic (e.g., 1% → 5% → 25% → 100%), and real production metrics (error rate, latency, saturation — the "RED"/"golden signals") are compared statistically between canary and baseline before each traffic increment. Tooling: Argo Rollouts, Flagger + Istio/Linkerd, Spinnaker's automated canary analysis (Kayenta). Pros: bounds blast radius to a small percentage of real users, catches regressions with real traffic patterns that staging can't reproduce, supports automatic rollback on metric regression. Cons: needs traffic-splitting infrastructure (service mesh/ingress), needs statistically meaningful analysis (naive canaries with too little traffic give false confidence), and takes longer end-to-end than blue-green.

**Rollback strategies.** Three levers, from fastest to slowest: (1) traffic-level rollback — flip the router/mesh weight back to the old version (fast, no redeploy); (2) GitOps revert — `git revert` the manifest/version bump and let the CD controller reconcile (minutes); (3) full redeploy of last-known-good image. Safety-critical systems should always prefer (1): keep the previous version's pods warm during rollout (don't scale them down until the canary is fully promoted) so rollback is a traffic-shift, not a rebuild.

### 3. Feature Flags

Feature flags **decouple deployment (code reaching production) from release (code being active for users)**. This is the single most important idea in modern release engineering: a deploy becomes low-risk because the new code path is dark (flagged off) by default, and "release" becomes a runtime, reversible, instant operation.

**LaunchDarkly model:** flags are evaluated per-request against a "context" (user ID, device, org, region, custom attributes). Targeting rules are an ordered list of clauses (e.g., `if org in ["beta-partners"] serve true`; `if percentage-rollout serve true to 10%` using a consistent hash of context key so the same user always gets the same variation). SDKs receive rule updates via a streaming connection and evaluate flags **locally, in-process**, against an in-memory cache — so evaluation is a microsecond in-memory lookup, not a network call per request, and flag data is still served (from last-known cache/default) if connectivity drops (source: LaunchDarkly architecture docs).

**AWS AppConfig model:** feature flags are JSON configuration profiles stored in AppConfig, deployed via a **deployment strategy** (percentage growth rate + bake time, e.g., linear 10%/hour), fetched by the AppConfig Lambda extension/agent and cached locally with short TTL polling. Rollback is automated via **CloudWatch alarms**: if an alarm tied to the deployment enters ALARM state during the bake window, AppConfig automatically halts and reverts to the prior configuration version — giving flag rollouts the same automated safety net as a canary code deployment.

**Patterns enabled by flags:** kill switches (instantly disable a misbehaving feature without a deploy), percentage/gradual rollouts independent of infra rollout, A/B experiments, per-tenant/per-region enablement, and "trunk-based development" (merge incomplete features behind a flag rather than long-lived feature branches). The operational cost is flag debt — stale flags must be retired or they become an untested combinatorial config surface.

### 4. Internal Developer Platforms (IDPs)

As microservice counts grow into the hundreds, the bottleneck stops being "can we build a pipeline" and becomes "can every team self-serve a correct, secure, observable pipeline without deep platform expertise." IDPs (Backstage, Port, Humanitec) solve this by providing:

- **Software/service catalog** — a single source of truth for what services exist, who owns them, their dependencies, on-call rotation, SLOs, and links to dashboards/runbooks/repos.
- **Golden path templates ("scaffolder")** — a new service is created from a vetted template that already wires up CI workflow, Dockerfile, Helm chart, monitoring dashboards, and on-call — so a team's first deploy is compliant by default rather than something a platform team audits after the fact.
- **TechDocs / documentation-as-code** co-located with the service.
- **Plugins that surface CI/CD status, canary rollout state, feature-flag state, cost, and security posture** directly on the service's catalog page, so an engineer doesn't need eight different tools' credentials to answer "is my service healthy and who owns the dependency it just broke."

Backstage (open-sourced by Spotify) is the dominant OSS IDP framework; Port is a commercial no-code alternative emphasizing configurable data models and self-service actions without writing plugins. The core value proposition of an IDP is turning platform capabilities (build, deploy, observe, secure) into paved-road, discoverable self-service — cutting cognitive load so hundreds of independent teams can ship safely without a central approval bottleneck.

---

## Case Study Solution: Uber-style Deployment Pipeline

### Problem Statement & Clarifying Requirements

Design the CI/CD and release platform for a ride-hailing company running **several thousand microservices** (dispatch, pricing/surge, ETA, driver location, payments, maps, notifications, supporting hundreds of engineering teams), each deploying **many times per day**, where some services are **safety- and revenue-critical** (dispatch matching, pricing, payments) and a bad deploy can strand riders, mis-price trips, or cause regional outages.

**Functional requirements**
- Every team can build, test, and deploy its own service independently, multiple times/day, without a central release manager.
- Support rolling, blue-green, and canary rollout strategies, selectable per service.
- Automatic, metrics-driven rollback with no human in the loop for the default case.
- Decouple "deployed" from "released" via feature flags, with per-city/per-rider-segment targeting (Uber operates market-by-market).
- Central service catalog so on-call/ownership/dependency data is discoverable during an incident.

**Non-functional requirements**
- Blast radius containment: a bad deploy of any one service should never affect >X% of trips in a region.
- Deploy pipeline itself must be highly available (release velocity of the whole company depends on it) and horizontally scalable to thousands of concurrent pipelines.
- Rollback latency: seconds to low minutes, not "redeploy from scratch."
- Auditability: every production change traceable to a commit + approver, for compliance and incident postmortems.
- Multi-region: services deploy independently per region/data center to avoid global blast radius.

### Capacity Estimation

- ~4,000 microservices, each deployed on average ~3×/day by its owning team (some services many times/hour during active development, others weekly) → **~12,000 deploys/day** company-wide, i.e., ~0.5–1 deploy/second sustained, with peaks during business hours far higher.
- Each deploy triggers a CI build (~5–15 min) and a canary rollout (~10–60 min bake, depending on service risk tier) → at any moment, plausibly **hundreds of pipelines and canary analyses in flight concurrently**.
- Feature flag evaluations: dispatch/pricing services evaluate flags on the hot path — at Uber's scale, tens of millions of trips/day translate to **hundreds of thousands of flag evaluations per second** at peak; this mandates in-process/in-memory flag evaluation (no synchronous network call per flag check), consistent with the LaunchDarkly/AppConfig local-cache model above.
- Service catalog: ~4,000 services × metadata (owner, on-call, SLO, dependency edges) — small in absolute data size (low GB), but read-heavy (every engineer, every incident) so it needs to be a fast, highly cached read path, not the write-path bottleneck.

### High-Level Architecture

```
Developer ── PR ──▶ GitHub Actions CI ──▶ Image Registry (signed, SHA-tagged)
                         │
                         ▼
                 Config/GitOps Repo (image tag bump)
                         │
                         ▼
              ┌─────────────────────────┐
              │   CD Controller (Argo)   │
              └─────────────────────────┘
                         │
       ┌─────────────────┼─────────────────────┐
       ▼                 ▼                     ▼
 Canary Controller   Feature Flag Service   Service Catalog / IDP
 (Argo Rollouts/      (per-flag targeting,   (Backstage-style:
  Flagger)             streamed to SDKs)      ownership, SLOs,
       │                                       on-call, deploy state)
       ▼
 Service Mesh (Istio) traffic split:
   99% → stable pods   1% → canary pods
       │
       ▼
 Metrics Pipeline (Prometheus/M3) ──▶ Automated Canary Analysis
       │                                   │
       ▼                                   ▼
 Alert on regression ──────────▶ Auto-rollback (shift traffic to 0%,
                                  revert GitOps commit)
```

Flow: a commit passes CI → image is built once, immutably → the CD controller reconciles the desired-state repo → the canary controller shifts a small percentage of mesh traffic to the new version → the metrics pipeline continuously compares canary vs. baseline (error rate, p99 latency, dispatch-match success rate) → on regression, traffic is shifted back to 0% and the GitOps commit is auto-reverted, all without human intervention (mirroring Uber's own μDeploy design, which explicitly favors "automatic rollback, often before all hosts have the new version," using I/O degradation, uncaught exceptions, HTTP error codes, and load as the automated signals — see Sources).

### API Design — Pipeline Stage Contract / Deployment Manifest

Instead of a request/response API, the contract is the schema each pipeline stage reads/writes so stages are decoupled and independently retriable.

```yaml
# deployment-manifest.yaml (written by CI, read by CD controller)
service: dispatch-matcher
version: git-sha-9f2c1a4
image: registry.uber.internal/dispatch-matcher:9f2c1a4
riskTier: safety-critical        # drives which rollout strategy is mandatory
rolloutStrategy:
  type: canary
  steps:
    - setWeight: 1
      pause: {duration: 5m}
    - analysis:
        metrics: [error_rate, p99_latency, match_success_rate]
        failureThreshold: 3      # consecutive failed checks => abort
    - setWeight: 10
      pause: {duration: 10m}
    - setWeight: 50
      pause: {duration: 15m}
    - setWeight: 100
approvals:
  required: true                 # safety-critical services need human sign-off past 10%
  approvers: [oncall-dispatch]
featureFlags:
  - key: new-matching-algorithm
    defaultVariation: control
rollback:
  auto: true
  onMetricRegression: revertGitCommit
```

### Data Model

**Deployment metadata (service catalog / audit store):**
```
deployments(
  deployment_id PK, service_id FK, git_sha, image_tag,
  region, risk_tier, strategy_type, initiated_by,
  status ENUM(pending, canarying, promoting, succeeded, rolled_back),
  started_at, finished_at, approver_id, rollback_reason
)
services(
  service_id PK, name, owning_team, oncall_rotation_id,
  repo_url, tier, slo_json, dependency_edges[]
)
```

**Feature flag config (LaunchDarkly/AppConfig-style):**
```
flags(
  flag_key PK, environment, default_variation,
  targeting_rules JSON,   -- ordered clauses: attribute, operator, values, serve
  rollout JSON,           -- {variation: weight} percentage buckets keyed by hashed context id
  version, updated_by, updated_at
)
```

**Canary metrics (time-series, read by the analysis engine):**
```
canary_metrics(
  deployment_id FK, timestamp, cohort ENUM(canary, baseline),
  metric_name, value, region
)
```
The analysis engine runs a statistical test (e.g., Mann-Whitney or simple threshold + z-score) comparing `canary` vs `baseline` cohorts per metric at each pause step before allowing `setWeight` to advance.

### Deep Dive

**Why canary over blue-green for the dispatch/pricing service.** Blue-green gives an instant, clean cutover, but it is an all-or-nothing bet: 100% of trips move to the new version the moment you flip, so any bug not caught in staging hits the entire fleet simultaneously — unacceptable for a service where a bug means mis-dispatched or mis-priced rides at national scale. Canary bounds the blast radius mathematically: at 1% traffic, a severe regression affects roughly 1 in 100 trips, for the few minutes it takes automated analysis to detect and roll back — and detection uses real production traffic patterns (real surge conditions, real geo-distribution) that a green environment's synthetic or shadow traffic can't fully replicate. The cost of canary — slower rollout, need for mesh-level traffic splitting and statistically rigorous analysis — is worth paying precisely because this service is safety-critical; a purely internal, low-traffic admin tool would be better served by blue-green's simplicity and instant rollback. In practice both are combined: canary for the gradual exposure/analysis, with the previous version's full fleet kept warm (not scaled down) so the final "abort" action is a blue-green-style instant traffic-weight flip back to the known-good version.

**How feature flags decouple deploy from release here.** The canary rollout answers "is this code safe to run in production" (infra risk); the feature flag answers "should this business logic be active for this rider/driver/city" (product/business risk) — and these are orthogonal. Uber can deploy `new-matching-algorithm` to 100% of dispatch-matcher instances (infra-safe, validated by canary analysis) while the flag keeps it dark for 99% of markets, then ramp the flag city-by-city independently of any further code deploys. If the algorithm misbehaves in one city, the fix is a flag flip (seconds, no deploy, no redeploy risk) rather than a rollback of running code. This also lets pricing/growth teams run controlled experiments (A/B) on the already-deployed, already-infra-validated code path.

**How an internal developer platform (Backstage-style catalog) helps hundreds of teams deploy independently and safely.** With thousands of services, the risk isn't just "will my deploy break," it's "do we even know who owns the dependency my service just started timing out on." The catalog encodes ownership, on-call, SLOs, and the dependency graph as structured, queryable data, so: (1) golden-path scaffolding means every new service is born with the correct CI workflow, risk-tier-appropriate rollout strategy, dashboards, and alerting already wired — a team never has to hand-roll a pipeline or "forget" a rollback policy; (2) during an incident, on-call for the failing downstream service is one click away instead of a Slack archaeology exercise; (3) platform-level policy (e.g., "safety-critical tier requires canary + manual approval past 10%") is enforced centrally in the template/pipeline-as-code, not by convincing 300 teams individually. This is what allows deploy velocity (thousands/day) and safety to coexist — the platform absorbs the correctness burden so individual teams only own their business logic.

### Trade-offs and Alternatives Considered

| Decision | Chosen | Alternative | Trade-off |
|---|---|---|---|
| Rollout for safety-critical services | Canary + kept-warm rollback | Pure blue-green | Canary bounds blast radius but is slower and needs mesh tooling; blue-green is instant but all-or-nothing |
| Rollout for low-risk internal services | Rolling update | Canary for everything | Canary for every service is operationally expensive (mesh + analysis infra) and unnecessary when blast radius is inherently small |
| Flag evaluation | In-process cache + streaming updates | Synchronous network call per evaluation | Network-per-eval adds latency/cost at hundreds of thousands of evals/sec and creates a single point of failure on the hot path |
| Config source of truth | GitOps (git as desired state) | CD tool holds mutable state only | GitOps gives free audit trail and trivial `git revert` rollback; costs some reconciliation latency vs. direct imperative deploys |
| Platform enforcement | Central IDP + golden-path templates | Per-team bespoke pipelines | Bespoke pipelines are more flexible short-term but don't scale governance/safety across hundreds of teams |

### How Real Systems Solve This

Uber's **μDeploy** (Micro Deploy) system deploys to a canary area first, using automated signals — I/O degradation, uncaught exceptions, HTTP error codes, throughput/load — to decide rollback without human review, explicitly favoring rolling back "long before all hosts have the new version" (Uber Engineering blog). Uber also runs **continuous deployment for large monorepos**, using controlled, staged rollout of monorepo-wide changes across many services to limit blast radius when a single commit touches shared code paths used by hundreds of services (Uber Engineering blog, "Controlling the Rollout of Large-Scale Monorepo Changes"). LaunchDarkly's architecture streams targeting-rule updates to SDKs that evaluate flags against an in-memory cache with default-value fallback on connectivity loss, keeping flag checks off the network hot path (LaunchDarkly docs). AWS AppConfig achieves the same decoupled, safe-rollout pattern for teams not running a full flag SDK, using deployment strategies (time/percentage-based growth) with CloudWatch-alarm-triggered automatic rollback. Backstage (built at Spotify, now a CNCF graduated project) is the reference implementation of the service-catalog/golden-path model described above, and is what companies operating at Uber-like microservice counts adopt (or build proprietary equivalents of) to keep hundreds of independent teams safely self-service.

## Sources

- [Uber Engineering's Micro Deploy: Deploying Daily with Confidence](https://eng.uber.com/micro-deploy-code/)
- [Controlling the Rollout of Large-Scale Monorepo Changes — Uber Engineering](https://www.uber.com/us/en/blog/controlling-the-rollout-of-large-scale-monorepo-changes/)
- [Continuous deployment for large monorepos — Uber Engineering](https://www.uber.com/us/en/blog/continuous-deployment/)
- [Continuous Integration and Deployment for Machine Learning Online Serving and Models — Uber Blog](https://www.uber.com/blog/continuous-integration-deployment-ml/)
- [LaunchDarkly architecture — LaunchDarkly Documentation](https://launchdarkly.com/docs/home/getting-started/architecture)
- [Flag evaluation rules in server-side SDKs — LaunchDarkly Documentation](https://launchdarkly.com/docs/sdk/concepts/flag-evaluation-rules)
- [Manage targeting rules — LaunchDarkly Documentation](https://launchdarkly.com/docs/home/flags/manage-rules)
- [Target with flags — LaunchDarkly Documentation](https://launchdarkly.com/docs/home/flags/target)
- [Deploying feature flags and configuration data in AWS AppConfig — AWS Documentation](https://docs.aws.amazon.com/appconfig/latest/userguide/deploying-feature-flags.html)
- [Implementing dynamic feature flags with AWS AppConfig on AWS Lambda — AWS Compute Blog](https://aws.amazon.com/blogs/compute/implementing-dynamic-feature-flags-with-aws-appconfig-on-aws-lambda/)
- [Backstage Software Catalog and Developer Platform — Demos](https://backstage.io/demos/)
- [What is Spotify Backstage and how does it work — getDX](https://getdx.com/blog/spotify-backstage/)
- [Backstage by Spotify — The Ultimate Guide — Roadie.io](https://roadie.io/backstage-spotify/)
