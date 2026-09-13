# Fast-Track: Module 18 - Containers & Kubernetes
**Core Concept:** Docker, K8s (Deployments, Pods, Services, HPA). GitOps (ArgoCD).
**Case Study:** YouTube Top-K Trending
- **Key Insight:** Use StatefulSets for partition-affinity in streaming workers. Scale via KEDA based on Kafka consumer lag, not just CPU. Count-Min Sketch for memory-efficient counting.
- **Takeaway:** CPU isn't always the best scaling metric for async workers; use queue depth / lag.
