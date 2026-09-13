# Case Study: Instagram on AWS (Media Upload)
- **Requirements:** Upload large images/videos without choking API servers.
- **Architecture:** S3 Presigned URLs + S3 Event Notifications + SNS/SQS.
- **Key Insight:** Never proxy media through EC2. Client requests a Presigned POST URL from the API, uploads directly to S3. S3 triggers an event to SNS, which fans out to SQS queues for thumbnailing and ML moderation.
