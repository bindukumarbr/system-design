# Module 18: Containers & Kubernetes

## Core Concepts

### Docker & Multi-Stage Builds
- **Layers:** Each command in a Dockerfile creates a cached layer. Put frequently changing commands (like copying source code) at the very bottom so you don't invalidate the cache for slow commands (like installing OS packages).
- **Multi-Stage Builds:** You don't want the Go compiler in your final production image. Use a `build` stage to compile the binary, then `COPY --from=build` into a tiny, scratch base image.

### Kubernetes Architecture
- **Control Plane:** `kube-apiserver` (API entry point), `etcd` (database of state), `kube-scheduler` (assigns pods to nodes), `kube-controller-manager` (maintains desired state).
- **Node:** The worker VM. Runs `kubelet`.
- **Pod:** Smallest unit. One or more containers sharing a network namespace.
- **Deployment:** Manages ReplicaSets and handles rolling updates.
- **Service:** A stable virtual IP that load balances across the ephemeral Pods.

### ConfigMaps & Secrets
- **ConfigMaps:** Environment variables and config files.
- **Secrets:** Base64 encoded sensitive data. *Note: Base64 is not encryption.* You must configure etcd encryption at rest or use a tool like External Secrets Operator.

### GitOps (ArgoCD & Flux)
- Instead of Jenkins running `kubectl apply`, a Git repo is the single source of truth. 
- ArgoCD lives inside the cluster, watches the Git repo, and pulls changes automatically. If someone manually deletes a Pod, ArgoCD puts it back.

---

## Case Study: YouTube Top-K Trending System
- **Requirements:** Compute the top trending videos by region in near real-time. High write throughput.
- **Estimations:** 115,000 views per second on average. 500,000 at peak.
- **Architecture:** 
  - **Ingestion:** API Gateway -> Kafka. Kafka partitions by region/category.
  - **Aggregation (Kubernetes):** StatefulSets running workers. Each worker consumes from Kafka and holds a **Count-Min Sketch** in memory.
  - **Serving Store:** Redis holding the top 100 lists.
- **Aha! Insights:**
  - **Count-Min Sketch:** A probabilistic data structure. You can't keep a hash map of 500 million videos in memory. Count-Min Sketch allows you to estimate the view count of any video using very little memory, with a small error bound.
  - **StatefulSets vs Deployments:** Because the aggregation workers hold the Count-Min Sketch in memory, if a Pod restarts, it would lose all its data. By using a StatefulSet + Persistent Volume, if `worker-0` crashes, K8s brings it back up and attaches the exact same disk, so it doesn't have to replay the entire Kafka topic from zero.
