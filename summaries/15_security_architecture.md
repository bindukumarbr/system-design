# Module 15: Security Architecture

## Core Concepts

### Defense in Depth & Zero Trust
- **Defense in Depth:** Layered security. If the firewall fails, the API Gateway still authenticates. If the Gateway fails, the Database still encrypts.
- **Zero Trust:** Being inside the VPC (internal network) means nothing. Every service-to-service call must prove its identity on every request.

### OAuth 2.0 & OIDC (Authorization Code Flow with PKCE)
- The modern standard for authentication.
- **PKCE (Proof Key for Code Exchange):** Prevents attackers from stealing the authorization code and trading it for an access token. The client generates a random verifier, hashes it (the challenge), and sends the challenge first. Later, it sends the raw verifier to prove it is the original requester.

### API Gateway Authentication
- Validates JWTs at the edge.
- Fetches the Identity Provider's (IdP) public keys from the JWKS endpoint (`/.well-known/jwks.json`) to verify the JWT's RS256 asymmetric signature.
- Validates `iss` (issuer), `aud` (audience), `exp` (expiry).

### mTLS (Mutual TLS)
- Standard TLS only verifies the server. mTLS verifies **both client and server**.
- Used for service-to-service communication. Each microservice gets a short-lived certificate (e.g., from Istio/Envoy sidecar).

### Dynamic Secrets (HashiCorp Vault / AWS Secrets Manager)
- Storing DB passwords in environment variables is bad.
- Vault generates **short-lived, dynamic credentials** (e.g., a Postgres user that self-destructs in 1 hour). A leaked credential is useless after a short time.

---

## Case Study: Facebook Post Search
- **Requirements:** Full-text search over posts. **Crucial:** Never return a post if the searcher doesn't have permissions (e.g., Audience is "Friends Only", and they just unfriended).
- **Estimations:** 300M new posts/day. 50M searches/day.
- **Architecture:** 
  - **API Gateway:** Terminates TLS, validates JWT, converts to internal short-lived JWT.
  - **Query Service:** Parses query, fetches the user's friend graph, fans out to shards.
  - **Index Shards:** Contains the post text + denormalized privacy metadata (`audience_type: FRIENDS`).
- **Aha! Insights:**
  - **Live Authorization (BOLA Prevention):** You cannot bake the list of a person's friends into the search index, because friendships change constantly. Instead, the Query Service retrieves the searcher's live friend graph, passes it to the shards, and the shards do an in-memory intersection check at query time.
  - **Filter-then-Verify:** The shards do a coarse pre-filter, but the Query Service does a final authoritative authorization check before returning the payload to the user. This prevents OWASP API1 (Broken Object Level Authorization).
