# Fast-Track: Module 17 - Cloud Architecture (AWS)
**Core Concept:** EC2/ECS/Lambda, S3, RDS, VPCs, ALB/NLB, SQS/SNS, Terraform.
**Case Study:** Instagram on AWS
- **Key Insight:** Never proxy large media files through your API. Use Presigned S3 POST URLs for direct client uploads, triggering async processing via S3 events -> SNS -> SQS.
- **Takeaway:** Leverage cloud-managed services for heavy lifting (storage, queues, CDNs).
