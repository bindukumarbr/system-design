# Module 18: Containers & Kubernetes

## Core Concepts

### Docker: images, layers, multi-stage builds

A Docker **image** is a read-only, layered filesystem plus metadata (entrypoint, env, exposed ports). Each instruction in a Dockerfile (`FROM`, `RUN`, `COPY`, `ADD`) produces a new **layer**, content-addressed by hash and cached: if a layer's inputs haven't changed, Docker reuses the cached layer instead of rebuilding it. This is why Dockerfile ordering matters — put rarely-changing steps (installing OS packages, dependency resolution) before frequently-changing steps (copying application source), so a source-code change only invalidates the last few layers, not the whole build. At runtime, a **container** adds a thin writable layer on top of the image's stacked read-only layers via a union filesystem (overlay2 on Linux).

**Multi-stage builds** solve the problem of bloated production images that carry compilers, build tools, and intermediate artifacts. A Dockerfile can declare multiple `FROM` stages; later stages selectively `COPY --from=<stage>` only the compiled artifact, discarding the build toolchain:

```dockerfile
FROM golang:1.22 AS build
WORKDIR /src
COPY . .
RUN go build -o /app ./cmd/server

FROM gcr.io/distroless/static-debian12
COPY --from=build /app /app
ENTRYPOINT ["/app"]
```

This yields a final image with no shell, no package manager, and a minimal attack surface — often tens of MB instead of 1GB+. Multi-stage builds are the standard pattern for compiled languages (Go, Rust, Java) and are also used to run tests or linters in an intermediate stage that never ships.

### Kubernetes architecture

Kubernetes (K8s) is a declarative control-plane for scheduling and reconciling containerized workloads. Core components:

- **Control plane**: `kube-apiserver` (the single entry point, validates and persists objects to etcd), `etcd` (distributed key-value store holding all cluster state), `kube-scheduler` (assigns unscheduled pods to nodes based on resource requests, affinity/anti-affinity, taints/tolerations), and `kube-controller-manager` (runs reconciliation loops — ReplicaSet controller, Node controller, etc. — each watching the API server and driving actual state toward desired state).
- **Node**: a worker machine (VM or bare metal) running `kubelet` (the agent that starts/stops containers per the PodSpec and reports node/pod status), `kube-proxy` (programs iptables/IPVS rules to implement Service virtual IPs), and a container runtime (containerd or CRI-O, since Docker Engine itself was deprecated as a runtime in favor of the CRI-standardized containerd).
- **Pod**: the smallest deployable unit — one or more containers sharing a network namespace (same IP, localhost between containers) and optionally storage volumes. Pods are ephemeral and disposable by design.
- **Deployment**: a controller that manages a **ReplicaSet**, which in turn maintains a target count of identical pod replicas. Deployments own the rolling-update algorithm, revision history, and rollback semantics (see below).
- **Service**: a stable virtual IP + DNS name that load-balances traffic across a dynamic set of pods matched by label selector, decoupling clients from pod churn. `ClusterIP` (internal only), `NodePort` (exposes a port on every node), `LoadBalancer` (provisions a cloud LB), and `Headless` (no VIP, returns pod IPs directly via DNS — used for stateful, per-instance addressing).

### ConfigMaps, Secrets, HPA, networking model, Helm

**ConfigMaps** hold non-sensitive configuration (env vars, config files) as key-value data, injected into pods via env vars, `envFrom`, or mounted volumes — decoupling image from configuration so the same image runs across environments. **Secrets** hold sensitive data (credentials, TLS keys, tokens); structurally similar to ConfigMaps but base64-encoded (not encrypted by default — encryption-at-rest for etcd and integration with an external secret store like AWS Secrets Manager, Vault, or Sealed Secrets/External Secrets Operator is required for real security) and mounted with tighter RBAC and tmpfs-backed volumes to avoid disk persistence.

**Horizontal Pod Autoscaler (HPA)** adjusts replica count based on observed metrics against a target. The classic form scales on CPU/memory utilization (via `metrics-server`); production systems commonly scale on **custom or external metrics** (queue depth, requests-per-second, Kafka consumer lag) exposed through the custom/external metrics APIs and an adapter (Prometheus Adapter, KEDA). HPA polls metrics every sync period (default 15s), computes `desiredReplicas = ceil(currentReplicas × (currentMetricValue / desiredMetricValue))`, and applies stabilization windows to prevent thrashing (separate scale-up/scale-down cooldowns configurable via `behavior`).

**Networking model**: Kubernetes mandates that every pod gets its own routable IP, all pods can reach all other pods without NAT, and a pod sees itself with the same IP other pods see it as — implemented by a CNI plugin (Calico, Cilium, AWS VPC CNI). Service-to-pod load balancing is handled by `kube-proxy` rewriting the Service VIP to a pod IP via iptables/IPVS DNAT rules, or bypassed entirely by eBPF-based dataplanes like Cilium for lower latency.

**Helm** is the package manager for Kubernetes: a **chart** is a templated bundle of manifests (Deployment, Service, ConfigMap, etc.) parameterized via `values.yaml`. `helm install`/`upgrade` renders templates and applies them, tracking release revisions so `helm rollback <release> <revision>` can revert an entire chart's rendered state atomically — the standard mechanism for versioning and promoting an application's full K8s footprint across environments.

### ECS on AWS: Fargate vs EC2

Amazon ECS is AWS's proprietary container orchestrator (an alternative to running your own Kubernetes / EKS). It supports two launch types:

- **EC2 launch type**: you provision and manage the EC2 instances (the "container instances") that run the ECS agent; ECS schedules tasks onto them. You control instance type, AMI, patching, and can pack multiple tasks per instance for higher density and lower cost, but you own capacity planning, scaling the underlying fleet (via Auto Scaling Groups / Capacity Providers), and patching.
- **Fargate launch type**: serverless — you specify CPU/memory per task and AWS provisions and manages the underlying compute, network, and isolation (each task gets its own micro-VM-level isolation via Firecracker). No node management, faster to adopt, better security isolation between tasks, but higher per-vCPU/GB cost, less control over instance types (no GPU support historically, spot-like variability), and cold-start/provisioning latency somewhat higher than a warm EC2 fleet.

The trade-off is fundamentally **operational overhead vs. cost efficiency and control**: EC2 launch type suits large, steady-state, cost-sensitive workloads with predictable density; Fargate suits spiky, heterogeneous, or ops-constrained workloads (batch jobs, low-traffic services, teams without dedicated infra staff) where eliminating instance management outweighs the price premium.

### GitOps: ArgoCD & Flux

GitOps treats a Git repository as the single source of truth for declarative infrastructure/application state; an in-cluster controller continuously reconciles live cluster state to match the repo, rather than a CI pipeline pushing changes imperatively.

- **ArgoCD**: pull-based, application-centric. An `Application` CR points at a Git path/Helm chart; ArgoCD's controller diffs live state vs. desired manifests and can auto-sync or require manual sync approval. It ships a UI/CLI for visualizing drift and diffs and supports **sync waves** and **PreSync/PostSync hooks** for ordering. Rollback = `argocd app rollback` to a prior Git-tracked revision, or simply `git revert` the desired-state commit and let auto-sync reconcile.
- **Flux**: a set of Kubernetes-native controllers (source-controller, kustomize-controller, helm-controller) built as CRDs, lighter-weight and more "invisible" (no bundled UI by default, composes with Kustomize/Helm natively), often preferred for pure infra-as-code pipelines and multi-tenancy via `Kustomization`/`HelmRelease` CRs per team/namespace.

Both support **progressive delivery** (canary/blue-green) via companion controllers — Argo Rollouts or Flagger — which gate promotion on live metrics (error rate, latency) queried from Prometheus, automatically rolling back if thresholds are breached.

### Rolling updates, rollback, image scanning

A Kubernetes Deployment's default update strategy is `RollingUpdate`, controlled by `maxUnavailable` and `maxSurge`: new-ReplicaSet pods are created incrementally while old-ReplicaSet pods are terminated, keeping the service available throughout. Readiness probes gate this — a new pod only counts as "up" once it passes its readiness check, so a broken new version stalls the rollout rather than taking down capacity. `kubectl rollout status` tracks progress; `kubectl rollout undo deployment/<name>` reverts to the previous ReplicaSet revision (Deployments retain a bounded revision history via `revisionHistoryLimit`), and `kubectl rollout undo --to-revision=N` targets a specific earlier revision.

**Container image scanning** is a required supply-chain control, typically wired into the CI pipeline before an image is pushed to a registry (Trivy, Grype, Snyk, or AWS ECR's native scanning) and again as an **admission control gate** in-cluster (OPA/Gatekeeper, Kyverno) that blocks deployment of images with unresolved critical CVEs or missing provenance (SBOM/cosign signature verification). Scanning at build time catches issues early and cheaply; the admission-time gate is the enforcement backstop that prevents an unscanned or previously-approved-but-now-CVE'd image from ever running.

## Case Study Solution: YouTube Top-K Trending System

### Problem statement & clarifying requirements

Design a system that continuously computes and serves the **top-K most-viewed/trending videos**, sliced by region and category, from a massive stream of view events, with results visible within seconds to a couple of minutes of the underlying activity.

**Functional requirements**
- Ingest view events (video_id, region, category, timestamp, user/session id) at very high throughput.
- Continuously maintain approximate view counts per video, windowed (e.g., trailing 1 hour, 24 hours).
- Serve top-K (K ≈ 10–50) videos for a given (region, category, window) with low read latency (p99 < 100ms).
- Support near-real-time freshness: new trends should surface within 1–2 minutes.

**Non-functional requirements**
- Massive write volume, read-heavy on a much smaller "trending" surface.
- High availability of the read path even if the aggregation layer lags or partially fails.
- Approximate correctness is acceptable (exact rank-1 precision is not required — this is explicitly a top-K/heavy-hitters problem, not an exact-counting problem).
- Horizontally scalable ingestion and aggregation as view volume grows (viral spikes).
- Multi-tenancy across regions/categories without one hot partition starving others.

### Capacity estimation

Assume 2 billion daily active users globally, with a conservative average of 5 video-view events triggered per user per day (view start, progress pings, completion) — real pipelines often log more granular engagement events, but we'll bound the estimate:

- Daily events ≈ 2B × 5 = 10B events/day.
- Average QPS ≈ 10B / 86,400s ≈ ~115,000 events/sec.
- Peak QPS (3–5x average for evening peak + regional overlap) ≈ 400,000–600,000 events/sec.
- Event size ≈ ~200 bytes (video_id, region, category, user/session id, timestamp, device metadata) → peak ingest bandwidth ≈ 500,000 × 200B ≈ 100 MB/s.
- Cardinality: assume ~500M distinct videos with any views in a rolling 24h window, but the "candidate trending set" per region/category is only ever a few thousand videos realistically competing for top-K — this cardinality gap is exactly why an approximate sketch (rather than an exact hash map of 500M counters) is the right tool.
- Storage for the serving layer is tiny: top-50 per (region × category) pair, maybe 250 regions × 20 categories = 5,000 keys × 50 entries × ~100 bytes ≈ 25MB — trivially cacheable in memory.

### High-level architecture

```
 [Client apps/players]
         │  view events
         ▼
 [Edge collectors / API gateway]  -- validate, stamp region, dedupe by session
         ▼
 [Kafka: topic "view-events", partitioned by (region, category, hash(video_id))]
         ▼
 [Streaming aggregation layer — Flink/Kafka Streams, one consumer-group worker
  per partition, running on Kubernetes]
     ├─ maintains Count-Min Sketch (or Space-Saving) per (region,category) shard
     ├─ maintains a bounded heap/TopK structure per shard, refreshed per micro-batch
     └─ emits periodic snapshots (every 5–15s) to the serving store
         ▼
 [Aggregator merge tier] -- merges per-partition sketches/top-K lists into
  global top-K per (region,category) using sketch-additivity property
         ▼
 [Serving store: Redis / DynamoDB, key = region:category:window]
         ▼
 [Top-K Read API] ── HPA-scaled Deployment on Kubernetes
         ▼
 [CDN / edge cache] → clients
```

Kafka absorbs the ingestion spike and decouples producers from the aggregation layer's processing rate — critical during a viral view-count surge. The **streaming aggregation layer** is stateful: each worker owns one or more Kafka partitions and keeps an in-memory **Count-Min Sketch** (a probabilistic frequency-counting structure: a 2D array of counters updated by k independent hash functions per event, giving frequency estimates with bounded over-count error and no false negatives) or a **Space-Saving** algorithm (which directly maintains a bounded set of the top-m candidates with count estimates, well-suited when you only need top-K rather than full frequency distribution). Because Count-Min Sketch is **linearly additive** (summing two sketches of the same dimensions yields the sketch of the merged stream), per-partition sketches can be periodically merged into a global sketch per (region, category) without re-processing raw events — this is the key trick that makes horizontal, shardable, near-real-time top-K approximation tractable at this volume.

### API design

```
GET /v1/trending?region=US&category=music&window=1h&k=25
→ 200 OK
{
  "region": "US",
  "category": "music",
  "window": "1h",
  "generated_at": "2026-09-05T14:32:00Z",
  "results": [
    { "rank": 1, "video_id": "abc123", "estimated_views": 812345, "score_confidence": "high" },
    { "rank": 2, "video_id": "def456", "estimated_views": 790112, "score_confidence": "high" },
    ...
  ]
}

GET /v1/trending/{video_id}/rank?region=US&category=music
→ returns a single video's current estimated rank/count (or "not in top-K")
```

The API is read-heavy, backed by an in-memory/cache-first store; `generated_at` communicates staleness to the client so UIs can show "updated Xs ago." K is capped (e.g. max 100) since only the head of the distribution is ever materialized.

### Data model

Serving-layer schema (per region/category/window key):

```
Key:   trending:{region}:{category}:{window}
Value: {
  generated_at: timestamp,
  entries: [ {video_id, estimated_count, rank}, ... ]   // sorted, bounded length K_max
}
```

Aggregation-layer state (per worker/shard, not directly queryable):

```
ShardState {
  partition_id,
  sketch: CountMinSketch(width=2000, depth=5),     // tunable epsilon/delta error bounds
  topk_heap: BoundedMinHeap(capacity=200),         // candidate superset, trimmed to K on read
  window_start, window_end
}
```

Window management uses a sliding/tumbling hybrid (e.g., 1-minute tumbling buckets combined at read time into a rolling 1h/24h view), which bounds memory and lets old buckets expire and be evicted from the sketch without recomputing from scratch.

### Deep dive: Kubernetes deployment topology

**Aggregation workers as a stateful topology.** Each worker owns exclusive ownership of one or more Kafka partitions and holds in-memory sketch state that is expensive to rebuild (must replay the partition from a checkpoint). This is modeled as a **StatefulSet**, not a Deployment, because:
- Pods get stable, predictable identity (`agg-worker-0`, `agg-worker-1`, ...) that maps deterministically to partition assignment — worker *N* always claims partitions `[N × p .. (N+1) × p)`, so a rescheduled pod resumes the same partition ownership instead of a random one, avoiding a full-state rebuild storm on every restart.
- Each pod gets its own PersistentVolumeClaim for local checkpoint state (sketch snapshots + Kafka consumer offsets), so a pod restarting on the same or a new node can restore from its own volume rather than replaying the entire partition from Kafka's retention window.
- Ordered, one-at-a-time rolling updates (`podManagementPolicy: OrderedReady` or `Parallel` with `updateStrategy: RollingUpdate` and `partition` staging) let you upgrade the ranking algorithm one shard at a time and watch for anomalies before proceeding.

**HPA on stream lag, not CPU.** CPU utilization is a poor proxy here — a worker can be CPU-idle while badly behind on its partition's offset during a viral spike. Instead:
- Deploy KEDA (Kubernetes Event-Driven Autoscaling) with a Kafka-lag scaler, or Prometheus Adapter exposing `kafka_consumergroup_lag` as a custom metric to HPA.
- HPA/KEDA target: scale out when `lag_per_partition > threshold` (e.g., 50,000 unconsumed messages) or `lag_time_estimate > 30s`. Since partition count bounds max useful parallelism (you cannot usefully run more consumers than partitions in one consumer group), scaling here also means **pre-provisioning enough Kafka partitions** for the target ceiling and using a StatefulSet replica count as the scaling dial up to that partition count — beyond that, re-partitioning the topic is a heavier operation planned separately, not an HPA reaction.
- Scale-down is deliberately conservative (long stabilization window, e.g. 5–10 minutes) because losing a worker mid-spike compounds lag; scale-up is aggressive (short window, large step) to react to viral spikes fast.

**Serving layer** (the stateless read API) is a plain Deployment with a standard CPU/RPS-based HPA, since it's stateless and horizontally trivial — this is the layer that actually needs to absorb bursty client read traffic.

**Config & secrets.** Kafka bootstrap servers, sketch dimensions (width/depth — tunable per environment for accuracy/memory trade-off), window sizes, and feature flags for ranking-algorithm variants live in a ConfigMap mounted as env vars, so a config-only change (e.g., adjusting sketch epsilon) doesn't require an image rebuild — only a pod restart via a ConfigMap-triggered rolling restart (commonly automated with Reloader or a checksum annotation on the pod template forcing a new revision). Kafka credentials, Redis/DynamoDB auth, and any TLS material are Secrets, sourced from AWS Secrets Manager via the External Secrets Operator so credentials are never committed to Git even under a GitOps model — only a *reference* to the secret's ARN is.

**GitOps rollout of a new ranking algorithm.** A new scoring/ranking algorithm version (e.g., switching from raw view-count to a decayed/EWMA-weighted score, or changing the sketch's hash family) is shipped as a new container image tag referenced in a Helm `values.yaml` change committed to the GitOps repo. ArgoCD (or Flux + Flagger) detects the diff and, rather than syncing all StatefulSet replicas at once:
1. Applies the change to a **canary subset** first — e.g., one partition-owning pod, or a shadow deployment that consumes a mirrored subset of the topic and writes to a shadow key namespace in the serving store, compared against production output without affecting client-facing results.
2. Argo Rollouts/Flagger gates promotion on automated analysis — comparing rank churn/estimated-count divergence between canary and baseline, plus standard health signals (consumer lag not growing, no crash loops) queried from Prometheus over a bake period (e.g., 15 minutes).
3. On success, promotes by updating the StatefulSet's pod template incrementally (partitioned rolling update across all shards); on failure, the controller automatically reverts the Git-tracked desired state (or the operator does `argocd app rollback` / `git revert`), and the StatefulSet's `OrderedReady` rollback restores prior-version pods shard-by-shard, each resuming from its own PVC checkpoint — so a bad ranking algorithm never fully replaces the previous one before it's caught, and rollback doesn't require replaying the Kafka topic from scratch.

**Image scanning** runs in CI on every merge to the ranking-worker image (Trivy/Grype against the built image, blocking the pipeline on critical CVEs), and again as a Kyverno admission-control policy in-cluster requiring a valid cosign signature and an attached SBOM before the image is allowed to run — closing the gap where a previously-scanned base image develops a newly disclosed CVE after being merged but before deployment.

### Trade-offs and alternatives considered

| Dimension | Kubernetes (this design) | ECS Fargate | ECS EC2 |
|---|---|---|---|
| Stateful shard affinity (StatefulSet + PVC) | First-class (StatefulSet, volumeClaimTemplates, stable network identity) | Not natively supported — EBS-backed persistence and stable identity require workarounds; Fargate tasks are inherently stateless-oriented | Possible via EBS + placement constraints, but manual |
| Autoscaling on custom stream-lag metric | Native via HPA/KEDA + custom metrics API | Via Application Auto Scaling + CloudWatch custom metric, workable but less flexible tooling ecosystem | Same as Fargate, plus you also scale the underlying ASG |
| Operational overhead | Highest — you run/upgrade the control plane (or pay for EKS) and CNI/ingress stack | Lowest — no node management at all | Medium — manage EC2 fleet, patching, capacity providers |
| Cost at sustained high throughput | Competitive if node bin-packing is tuned | Highest per-vCPU cost at sustained scale | Lowest at high, steady utilization (reserved/spot EC2) |
| GitOps/portability | Best — ArgoCD/Flux, portable across clouds | AWS-specific; no equivalent open GitOps-native primitive as mature | AWS-specific |

For this workload — long-lived stateful stream-processing shards with partition-affinity, needing fine-grained custom-metric autoscaling and cross-environment portability — **Kubernetes with StatefulSets** is the strongest fit. **ECS Fargate** is attractive for the lightweight, stateless serving-API tier alone (fewer ops, fast to scale, no node patching) if the org wants to avoid running a full K8s cluster just for that tier — a hybrid approach (Fargate for the read API, an existing shared EKS cluster for the stateful aggregation tier) is a realistic middle ground many teams choose. **ECS EC2** wins purely on steady-state cost if the org already has deep EC2/ASG operational maturity and no cross-cloud portability requirement.

### How real systems solve this

Real-world trending/ranking systems at this scale converge on the same core pattern: **shard by key, approximate with a sketch, merge additively, materialize only the head of the ranking.** YouTube's own trending page (and similarly, Twitter's historical "trending topics") is understood to be computed via streaming pipelines (historically Flume/MapReduce-based batch, evolving to streaming systems) that bucket engagement counts over sliding time windows and apply exponential-decay weighting so a video's rank decays as its view velocity cools, rather than trending being pure cumulative view count — this is why a video with a sudden spike outranks an older video with a larger total count. The general "Top-K / heavy-hitters" problem is a well-studied space: the Count-Min Sketch (Cormode & Muthukrishnan) and Space-Saving algorithm are the standard building blocks referenced in systems-design treatments of this exact problem, and open-source streaming frameworks (Apache Flink's windowed aggregations, Kafka Streams' `TopicNameExtractor` + state stores) provide the primitives — a local RocksDB-backed state store per task with periodic changelog-topic backup — that map directly onto the PVC-backed StatefulSet checkpointing strategy described above.

## Sources

- [Kubernetes: Horizontal Pod Autoscaling (GKE)](https://docs.cloud.google.com/kubernetes-engine/docs/concepts/horizontalpodautoscaler)
- [Optimize Pod autoscaling based on metrics](https://docs.cloud.google.com/kubernetes-engine/docs/tutorials/autoscaling-metrics)
- [Horizontal Pod Autoscaling with Custom Metrics — Pixie Labs](https://blog.px.dev/autoscaling-custom-k8s-metric/)
- [Amazon ECS vs AWS Fargate: Complete Guide to Launch Types](https://towardsthecloud.com/blog/amazon-ecs-vs-aws-fargate)
- [AWS Fargate vs. Amazon EC2: Launch Options for AWS ECS](https://www.stormit.cloud/blog/aws-fargate-vs-ec2/)
- [Fargate vs EC2: Choosing the Right Launch Type for ECS Workloads](https://rambunct.com/fargate-vs-ec2-choosing-the-right-launch-type-for-your-ecs-workloads/)
- [Comparing Amazon ECS Launch Types: EC2 vs Fargate](https://dev.to/lumigo/comparing-amazon-ecs-launch-types-ec2-vs-fargate-7c4)
- [Approximate Heavy Hitters and the Count-Min Sketch (Stanford CS168)](https://web.stanford.edu/class/cs168/l/l2.pdf)
- [Count-Min Sketch — Heavy Hitters Problem lecture notes (Rice COMP480/580)](https://www.cs.rice.edu/~as143/COMP480_580_Fall22/scribe/Lec9.pdf)
- [Top K: Choosing Optimal Count-Min Sketch Parameters](https://dzone.com/articles/top-k-count-min-sketch-configuration)
- [Top K / Trending System Design — System Design School](https://systemdesignschool.io/problems/topk/solution)
- [System Design Interview - Top K Problem - Heavy Hitters](https://serhatgiydiren.com/system-design-interview-top-k-problem-heavy-hitters/)
- [ArgoCD vs FluxCD: 2026 Comparative Analysis](https://dev.to/mechcloud_academy/the-gitops-standard-in-2026-a-comparative-research-analysis-of-argocd-and-fluxcd-46d8)
- [Argo CD vs Flux: Identifying the Superior GitOps Tool — Wallarm](https://www.wallarm.com/cloud-native-products-101/argo-cd-vs-flux-gitops-tools)
- [GitOps in 2026: ArgoCD vs Flux — Definitive Comparison](https://devstarsj.github.io/devops/kubernetes/gitops/2026/05/25/gitops-argocd-vs-flux-kubernetes-cd-comparison-2026/)
