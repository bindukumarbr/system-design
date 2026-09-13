# Module 17: Cloud Architecture (AWS + Multi-Cloud)

## Core Concepts

### Compute (EC2 vs ECS vs Lambda)
- **EC2:** Raw VMs. Use when you need OS-level control, GPU, or custom kernel tuning. You manage patching.
- **ECS (Fargate):** Serverless containers. AWS manages the underlying hosts. Best default for APIs and microservices. No capacity planning needed.
- **Lambda:** Event-driven, maximum 15-minute runtime. Billed per millisecond. Best for bursty, unpredictable traffic, or glue code. *Bad for long-running processes.*

### Storage (S3 vs RDS vs ElastiCache)
- **S3:** Object storage. Infinite scale, cheap. Good for images, videos, backups.
- **RDS / Aurora:** Relational SQL. Strong consistency, ACID transactions. Good for user data, financial records.
- **ElastiCache (Redis):** Sub-millisecond read cache. Sits in front of RDS.

### VPC Networking
- **Public Subnet:** Has an Internet Gateway (IGW). Hosts Load Balancers.
- **Private Subnet:** No direct internet access. Uses a NAT Gateway for outbound traffic. Hosts the Application Servers and Databases.
- **Security Groups:** Stateful firewalls attached to instances. Return traffic is automatically allowed. This is what you use for 99% of rules.
- **NACLs:** Stateless firewalls attached to subnets. Return traffic must be explicitly allowed. Used mostly as a coarse safety net.

### Load Balancers (ALB vs NLB)
- **ALB (Application Load Balancer):** Layer 7. Can route based on URL path (`/api` vs `/images`). Terminates TLS. The standard for web traffic.
- **NLB (Network Load Balancer):** Layer 4. Ultra-low latency. Just forwards TCP packets. Gives you a static IP.

### Asynchronous Communication (SQS)
- **Queue-based load leveling:** A burst of 10,000 requests goes into SQS. Downstream workers pull from it at their own pace, preventing the database from crashing.
- **Visibility Timeout:** When a worker pulls a message, it becomes hidden from other workers. If the worker crashes and doesn't delete it, the timeout expires and the message reappears for another worker to try.

---

## Case Study: Instagram-style Media Platform
- **Requirements:** Users upload photos/videos. View a chronological feed of followed users. Low latency global delivery.
- **Estimations:** 100M Daily Active Users. Read-heavy (100:1 reads to writes).
- **Architecture:** 
  - **Upload Path:** API returns an S3 *Presigned URL*. The mobile app uploads the 50MB video *directly to S3*, skipping the API tier entirely. 
  - **Async Processing:** S3 triggers an event to SNS -> SQS. Lambda workers resize the image and run moderation.
  - **Feed Storage:** A Redis Sorted Set acts as a fan-out cache for the feed.
- **Aha! Insights:**
  - **Presigned URLs:** Never proxy massive file uploads through your API servers (EC2/ECS). It wastes expensive compute memory and network bandwidth. Hand the client a presigned URL and let S3 handle the heavy lifting.
  - **Fan-Out on Write vs Read:** For normal users, Fan-Out on Write (push to all followers' feeds when a post is created) works well. But for celebrities (e.g., Selena Gomez with 400M followers), Fan-Out on Write would take hours. You must use a Hybrid approach (Fan-Out on Read for celebrities).
