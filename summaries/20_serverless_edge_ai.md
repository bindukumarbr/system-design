# Module 20: Serverless, Edge & AI Systems

## Core Concepts

### Lambda Cold Starts
- **The Problem:** When a Lambda hasn't been called in a while, AWS kills the container. The next request must wait for AWS to download the code, boot the runtime, and run init code (takes seconds).
- **The Fix:**
  - *Provisioned Concurrency:* Pay AWS to keep N instances permanently warm.
  - *SnapStart:* AWS takes a microVM snapshot of the initialized memory. Next boot resumes from the snapshot (90% faster).

### S3 Presigned URLs
- Generate a cryptographically signed URL that grants a client temporary permission (e.g., 5 mins) to directly upload/download a specific object to/from S3, bypassing your backend servers.

### Edge Computing (Cloudflare Workers vs Lambda@Edge)
- Run code in Points of Presence (PoPs) physically close to the user.
- **Cloudflare Workers:** Runs on V8 isolates. 0ms cold starts. Great for routing, auth checks, header manipulation.
- *Rule:* Never put authoritative writes (e.g., deducting money from an account) at the Edge. Edge is for stateless routing and cache.

### RAG (Retrieval-Augmented Generation) Architecture
- Stops LLM hallucinations by injecting facts into the prompt.
- **Flow:** 
  1. Chunk private documents -> Embed into Vectors -> Store in Vector DB (Pinecone, pgvector).
  2. User asks question -> Embed question -> Search Vector DB for Top-K similar chunks.
  3. Combine Top-K chunks + User Question into a Prompt -> Send to LLM.

---

## Case Study: Robinhood-style Trading Platform
- **Requirements:** Stream real-time market quotes. Execute buy/sell orders. AI Portfolio Assistant.
- **Estimations:** 500K concurrent connections streaming quotes at market open. Millions of messages/sec.
- **Architecture:** 
  - **Quotes (Read):** Exchange -> Kafka -> Fan-out Service -> Cloudflare Edge -> User WebSockets.
  - **Orders (Write):** API Gateway -> Lambda (Provisioned Concurrency) for initial validation/idempotency -> SQS -> Dedicated Long-Running Matching Engine -> RDS Multi-AZ.
  - **AI Assistant:** RAG orchestrator Lambda -> queries pgvector for statement chunks AND queries RDS replica for exact current account balance.
- **Aha! Insights:**
  - **Mixing Serverless and Servers:** The initial API Gateway + Lambda is great for auth and idempotency checking (rejecting double-clicks). BUT the actual Order Matching Engine is a dedicated, long-running service because a cold start during market open could cost users thousands of dollars in slippage.
  - **RAG Fact Checking:** LLMs are bad at math. The orchestrator uses vector search to find *context*, but it queries the standard SQL read replica to get the *exact account balance*, injecting the hard SQL numbers into the prompt so the LLM doesn't invent a portfolio value.
