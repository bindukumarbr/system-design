# Module 15: Security Architecture

## Core Concepts

### Defense in Depth (Layered Security Model)

Defense in depth assumes any single control will eventually fail, so security is layered rather than pinned on one perimeter. A typical layered stack for a backend system:

1. **Network edge** — WAF, DDoS protection, TLS termination, IP allow-listing.
2. **API gateway** — authentication (OAuth2/OIDC token validation), rate limiting, schema validation.
3. **Service mesh / transport** — mTLS between services, network policies (namespace/segment isolation).
4. **Application layer** — input validation, output encoding, authorization checks per-request (not just per-endpoint).
5. **Data layer** — encryption at rest, field-level encryption for sensitive data, least-privilege DB roles, row-level security.
6. **Secrets and identity** — short-lived credentials, dynamic secrets, workload identity instead of static keys.
7. **Observability and response** — audit logging, anomaly detection, intrusion detection, incident response runbooks.

The interview-relevant takeaway: a breach at any one layer (a leaked JWT, a compromised pod) should not cascade into full data exposure, because each downstream layer independently re-verifies identity and authorization.

### Zero Trust Architecture

Zero trust replaces the old model of "trusted internal network, untrusted internet" with "never trust, always verify" for every request, regardless of network location. Core principles:

- **No implicit trust from network location.** Being inside the VPC or cluster grants no privilege by itself.
- **Verify explicitly, per request.** Every call — including service-to-service — carries and validates identity (mTLS certs, signed tokens) and is authorized against policy at the time of the call.
- **Least privilege, just-in-time access.** Identities get the minimum scope needed, often with time-bounded, dynamically issued credentials rather than long-lived static ones.
- **Assume breach.** Design as if an attacker is already inside the network; segment aggressively (microsegmentation) so lateral movement is limited, and log everything for detection.
- **Continuous verification, not one-time login.** Session risk is re-evaluated (device posture, token expiry, anomaly signals), not granted once and trusted forever.

In practice this is implemented via a combination of an identity provider (IdP) issuing short-lived tokens, an API gateway/service mesh enforcing per-request auth, and mTLS for workload identity — the exact building blocks covered below.

### OAuth 2.0 and OIDC — Authorization Code Flow with PKCE

**OAuth 2.0** is an authorization framework: it lets a client obtain a token representing delegated access to a resource, without ever seeing the user's password. **OIDC (OpenID Connect)** is an identity layer on top of OAuth2 that adds the **ID token** (a JWT asserting who the user is) alongside the OAuth **access token** (which grants access to a resource).

The **Authorization Code flow with PKCE** (RFC 7636) is the current best-practice flow for browser SPAs, mobile apps, and increasingly server-side web apps too, because it protects a public client (one that cannot hold a secret) against interception of the authorization code.

Step by step:

1. **Client generates a PKCE pair.** It creates a random `code_verifier` (43–128 char random string) and derives `code_challenge = BASE64URL(SHA256(code_verifier))`.
2. **Authorization request.** The client redirects the user's browser to the authorization server's `/authorize` endpoint with `response_type=code`, `client_id`, `redirect_uri`, `scope`, `state` (CSRF protection), and `code_challenge` + `code_challenge_method=S256`.
3. **User authenticates and consents** at the authorization server (the IdP — e.g., Okta, Auth0, Google). Credentials never touch the client application.
4. **Authorization server redirects back** to `redirect_uri` with a short-lived, single-use `authorization code` and the original `state`.
5. **Token exchange.** The client calls the `/token` endpoint with the `code`, `redirect_uri`, `client_id`, and the original `code_verifier` (not the challenge).
6. **Authorization server verifies** that `SHA256(code_verifier) == code_challenge` sent in step 2. Only then does it issue the **access token** (and, for OIDC, an **ID token**, and optionally a **refresh token**).
7. **Client calls the resource server / API** with `Authorization: Bearer <access_token>`.

**Why PKCE matters:** without it, if an attacker intercepts the authorization code (e.g., via a malicious app registering the same custom URL scheme, or a compromised network log), they could redeem it for tokens themselves, because a public client has no client secret to prove it's the legitimate requester. PKCE binds the code to whoever generated the original `code_verifier` — the attacker has the code but not the verifier, so the exchange fails. This closes the authorization-code-interception attack described in RFC 7636 and is now recommended for all clients, not just mobile/SPA.

### API Gateway Authentication — JWT Validation at the Edge

Most systems terminate authentication at the API gateway (Kong, Envoy/Istio ingress, AWS API Gateway, custom Nginx/Lua) rather than in every microservice. The gateway performs:

- **Signature verification.** The JWT's header names an algorithm (e.g., `RS256`) and a `kid` (key ID); the gateway fetches the IdP's public keys from its **JWKS endpoint** (`/.well-known/jwks.json`), selects the matching key, and verifies the signature. This proves the token was issued by the trusted IdP and hasn't been tampered with. RS256 (asymmetric) is strongly preferred over HS256 for this — HS256 requires the gateway to hold the same secret used to sign, which doesn't scale across many verifying services.
- **Claims validation.** Check `iss` (issuer matches expected IdP), `aud` (audience matches this API), `exp`/`nbf` (not expired / not yet valid), and any custom claims (`scope`, `roles`, `sub`).
- **Expiry and clock skew.** Access tokens are deliberately short-lived (minutes to ~1 hour) to limit the blast radius of a leaked token; refresh tokens (longer-lived, revocable) are used to mint new access tokens without re-authenticating the user.
- **Downstream propagation.** After validation, the gateway typically strips or re-signs the token and forwards identity context (user ID, scopes) to internal services via trusted headers or a re-issued internal JWT — internal services then trust the gateway (or re-verify via mTLS-authenticated internal token) rather than each independently talking to the public IdP.

This "validate once at the edge" pattern reduces latency (no per-service round trip to the IdP) but means the gateway is a critical trust boundary — its key rotation, clock sync, and revocation-checking (or lack thereof, since JWTs are typically not checked against a revocation list) must be handled carefully.

### Secrets Management — Vault and AWS Secrets Manager

Static, long-lived credentials (DB passwords baked into config, IAM access keys committed to a repo) are one of the most common breach vectors. Modern secrets management addresses this with:

- **Centralized storage with access policy.** HashiCorp Vault and AWS Secrets Manager both store secrets encrypted at rest, gate access via IAM/ACL policy, and provide an audit log of every read.
- **Dynamic secrets.** Vault's database and AWS secrets engines can generate **short-lived, unique credentials on demand** for each client/lease — e.g., a service requests DB access, Vault creates a temporary Postgres user with a TTL, and automatically revokes it on lease expiry. This means no shared static DB password exists at all; a leaked lease is useless after minutes.
- **Automatic rotation.** AWS Secrets Manager can rotate secrets (e.g., RDS master password) on a schedule via a Lambda rotation function, updating the secret and the underlying resource together, with zero application downtime if the app pulls credentials at connection time rather than caching them forever.
- **Workload identity instead of static keys.** Vault's AWS/Kubernetes auth methods and AWS's IAM roles for service accounts (IRSA) let a workload authenticate using its platform-native identity (an EC2 instance profile, a K8s service account token) rather than an embedded API key — removing another class of static secret.
- **Encryption as a service.** Vault's Transit engine lets applications encrypt/decrypt without ever handling the raw encryption key.

### mTLS for Service-to-Service Authentication

Mutual TLS extends standard TLS (where only the server presents a certificate) so that **both sides present and verify certificates**. In a zero-trust internal network:

- Each service is issued a short-lived X.509 certificate (often by a service mesh like Istio/Linkerd or a Vault PKI secrets engine), identifying it (e.g., SPIFFE ID `spiffe://cluster.local/ns/payments/sa/api`).
- On every internal connection, both client and server validate each other's certificate against a shared CA, proving workload identity cryptographically — not just "this traffic came from inside the VPC."
- This gives **encryption in transit** and **strong service identity** simultaneously, enabling fine-grained authorization ("service A may call service B's `/charge` endpoint") enforced at the mesh sidecar layer, independent of network topology.
- Certificates typically rotate automatically (hours, not months), so a compromised cert has a small window of validity — the dynamic-secrets philosophy applied to identity rather than passwords.

### OWASP API Security Top 10 (2023)

1. **API1: Broken Object Level Authorization (BOLA)** — a user can access another user's object by manipulating an ID, because the API checks authentication but not per-object ownership.
2. **API2: Broken Authentication** — weak token handling, credential stuffing, missing rate limits on login, or flawed token validation.
3. **API3: Broken Object Property Level Authorization** — excessive data exposure or mass assignment: an endpoint returns or accepts fields the caller shouldn't see/set.
4. **API4: Unrestricted Resource Consumption** — no limits on request size, rate, or cost, enabling DoS or excessive infrastructure spend.
5. **API5: Broken Function Level Authorization** — a regular user can call an admin-only endpoint because role checks are missing or inconsistent.
6. **API6: Unrestricted Access to Sensitive Business Flows** — automatable abuse of a legitimate flow (e.g., bulk-buying limited inventory) with no bot/abuse protection.
7. **API7: Server Side Request Forgery (SSRF)** — the API fetches a user-supplied URL server-side without validation, allowing internal network access.
8. **API8: Security Misconfiguration** — verbose errors, open CORS, default credentials, missing security headers, unpatched components.
9. **API9: Improper Inventory Management** — undocumented/zombie API versions or endpoints (e.g., a deprecated internal API left reachable) without the same scrutiny as the current one.
10. **API10: Unsafe Consumption of APIs** — trusting data from third-party/upstream APIs without the same validation applied to user input.

## Case Study Solution: Facebook Post Search

### Problem Statement & Clarifying Requirements

Design a search system over user-generated posts (status updates, photos with captions, comments) that lets a user search for posts by keyword, and **must never return a post the searching user is not authorized to see**, based on that post's audience setting at post time (Public, Friends, Friends-of-friends, Custom list, Only Me) — and that setting can change or the audience relationship (friendship) can change after the post was created.

**Functional requirements:**
- Full-text/keyword search over posts (and comments) with ranking by relevance and recency.
- Enforce the post's current privacy/audience settings at query time, not just at index time.
- Support pagination, basic filters (date range, author).

**Non-functional requirements:**
- Low query latency: p99 < 200-300 ms for search results.
- High write/index throughput: hundreds of millions of new posts/comments per day.
- Strong consistency on privacy decisions (never leak a private post) — availability/staleness can be relaxed for ranking freshness, but never for access control.
- Horizontal scalability of both index and query serving.

### Capacity Estimation

- Assume ~500M daily active users, ~10% post per day → 50M posts/day; comments ~5x posts → 250M comments/day. Total ~300M new searchable documents/day ≈ 3,500 writes/sec average, bursty to ~15–20K/sec at peak.
- Average document (post + metadata + ACL) ≈ 2 KB → ~600 GB/day of new index data; with retention/compaction and replication factor 3, multi-PB steady-state index size for a multi-year corpus, sharded across thousands of index nodes.
- Search QPS: assume 500M DAU, 5% issue a search per day, ~2 queries per search session → ~50M queries/day ≈ 580 QPS average, peak ~5-10x ≈ 3,000-6,000 QPS.
- Each query touches many shards (fan-out) and must run a per-candidate ACL check — this filtering cost is the dominant latency risk and drives the architecture below.

### High-Level Architecture

```
                      ┌───────────────────────┐
 Client (web/mobile) ─▶  API Gateway            │  - TLS termination
                      │  - OIDC/OAuth2 login    │  - JWT signature+claims validation
                      │  - Rate limiting, WAF   │  - mTLS to internal services
                      └──────────┬──────────────┘
                                 │ internal JWT (short-lived) over mTLS
                                 ▼
                      ┌───────────────────────┐
                      │   Query Service        │
                      │  - parses query        │
                      │  - fetches user's      │
                      │    friend/group graph  │──────▶ Social Graph Service
                      │  - fans out to shards  │        (mTLS)
                      └──────────┬──────────────┘
                                 │ mTLS
                    ┌────────────┼─────────────┐
                    ▼            ▼             ▼
              Index Shard 1  Index Shard 2 ... Index Shard N
              (posts + inline ACL metadata; per-shard privacy
               pre-filter, then merge/rank at Query Service)

 Ingestion side (async):
 Post Service --(post created/edited/privacy changed event)--> Message Queue (Kafka)
        │
        ▼
 Indexing Pipeline (consumers) --tokenize, extract ACL snapshot--> Index Shards
        │
        └── on privacy change: emits re-index / ACL-update event (near-real-time)

 Secrets: Vault / AWS Secrets Manager issues DB creds, index-service
 credentials, and signing keys to Query Service & Indexing Pipeline
 (dynamic, short-lived, auto-rotated) — no static passwords in config.
```

### API Design

```
GET /v1/search/posts?q=<query>&cursor=<opaque>&limit=25&filter.author=<id>&filter.since=<ts>
Host: api.example.com
Authorization: Bearer <OAuth2/OIDC access token, JWT, RS256>
```

Response:
```json
{
  "results": [
    {
      "post_id": "p_9182...",
      "author_id": "u_5521...",
      "snippet": "...highlighted matching text...",
      "created_at": "2026-08-01T12:00:00Z",
      "score": 0.87
    }
  ],
  "next_cursor": "opaque-token"
}
```
The `Authorization` header carries the OIDC-issued access token (`aud` = search API, `scope=search:read`, short expiry, e.g., 15 min). The gateway validates it before any request reaches the Query Service; the Query Service additionally receives the authenticated `sub` (user ID) via a trusted internal header/re-signed internal token so it can do privacy filtering.

### Data Model (Search Index Schema)

Each indexed document carries denormalized ACL metadata alongside searchable content, so privacy filtering can happen inline during the search, without a separate blocking join per result:

```json
{
  "post_id": "p_9182...",
  "author_id": "u_5521...",
  "text_tokens": ["hiking", "yosemite", "weekend"],
  "created_at": 1754049600,
  "privacy": {
    "audience_type": "FRIENDS | PUBLIC | CUSTOM_LIST | ONLY_ME | FRIENDS_OF_FRIENDS",
    "allowed_list_id": "list_772...",     // for CUSTOM_LIST
    "denied_user_ids": ["u_991..."],       // explicit excludes ("all friends except...")
    "acl_version": 4                       // bumped whenever privacy changes
  },
  "engagement_score": 128,
  "deleted": false
}
```

Design notes: `audience_type=FRIENDS` does **not** enumerate the friend list in the document (friend graphs are large and change constantly) — instead the shard-level filter evaluates `is_friend(searching_user, author_id)` against a fast in-memory/graph-cache lookup, keeping the index document small and stable while the actual membership check happens at query time against current data.

### Deep Dive

**1. Authentication (OAuth2/OIDC + JWT at the gateway).** The user authenticates once via the OIDC authorization-code-with-PKCE flow against the identity provider, receiving a short-lived access token (JWT, RS256-signed). Every search request presents this as a Bearer token. The API gateway fetches the IdP's JWKS, verifies the signature, checks `iss`, `aud=search-api`, `exp`, and `scope=search:read`. Only a request with a valid, unexpired, correctly-scoped token reaches the Query Service — this establishes *who is asking*, but says nothing yet about *what they're allowed to see*, which is a separate, per-document decision made downstream.

**2. Privacy-aware filtering at query time, not just ingest.** This is the crux of the design. If filtering only happened at indexing time (e.g., baking a static ACL into the document once), the index would go stale the moment a friendship ends, a post's audience is edited, or someone leaves a custom list — silently leaking posts that should now be hidden, or hiding posts that should now be visible. Instead:
   - The indexing pipeline listens to a Kafka topic of `post.privacy_changed` and `friendship.changed` events and updates `acl_version`/audience metadata in near-real-time (seconds), so the *document's own* privacy fields stay fresh.
   - Critically, group-membership style checks (`FRIENDS`, `FRIENDS_OF_FRIENDS`, `CUSTOM_LIST`) are **not baked into the document as a snapshot of a group at post time** — the Query Service (or a colocated shard-level filter) evaluates them live against the current Social Graph Service for every candidate result before it's returned. This is effectively a live authorization check per document, similar to attribute-based access control (ABAC): `allow(viewer, post) = f(post.privacy, graph(viewer, post.author), current_time)`.
   - To keep this fast at scale, each shard does a coarse pre-filter locally (drop `ONLY_ME` posts unless viewer==author; drop `CUSTOM_LIST` unless viewer is cached as a list member) using a fast local cache of the viewer's social graph (fetched once per query, TTL-cached for seconds), then the Query Service does a final authoritative check on the merged top-K results before returning them — trading a slightly larger initial candidate set for a small, cheap final-authorization pass rather than authorizing every document in the full corpus.
   - This "filter-then-verify" two-stage approach mirrors OWASP's guidance against **Broken Object Level Authorization (API1)**: never trust that a document being retrievable from the index implies the requester is allowed to see it — authorization is re-evaluated on the response path, per object, on every request.

**3. mTLS usage.** All internal hops — gateway → Query Service, Query Service → Social Graph Service, Query Service → Index Shards, Indexing Pipeline → Index Shards — use mutual TLS with short-lived certificates issued by an internal PKI (e.g., Vault's PKI secrets engine or a service mesh's built-in CA). This authenticates *which service* is calling (not just that traffic originated inside the network), encrypts data in transit (post content and privacy metadata are sensitive), and lets policy be expressed as "only the Query Service identity may call Index Shard read endpoints," containing blast radius if any one service is compromised.

**4. Secrets management.** The Indexing Pipeline and Query Service need credentials to reach the post datastore, the message queue, and the index shards. Rather than static passwords in config or environment variables, they authenticate to Vault (or AWS Secrets Manager, if AWS-native) using their workload identity (Kubernetes service account / IAM role), and receive **dynamic, short-lived credentials** — e.g., a scoped DB user with a 1-hour TTL — that are automatically rotated and revoked. Signing keys used to re-issue internal tokens are similarly stored and rotated via Vault's Transit engine rather than hardcoded.

### Trade-offs and Alternatives Considered

- **Bake ACLs into the index at write time vs. live check at query time.** Pure write-time baking is faster per query but goes stale and risks leaking privacy changes; pure query-time graph lookups for every candidate are the most correct but too slow at fan-out scale. The chosen hybrid (denormalized coarse metadata + live final-stage check on the merged top-K) balances latency and correctness.
- **Separate index per privacy tier vs. single index with inline metadata.** Splitting into "public index" vs. "private index" simplifies some queries but complicates ranking/merging across tiers and still requires live friend-graph checks for `FRIENDS` audiences — not much of a win; a single index with rich per-document ACL metadata was chosen for simpler operations.
- **Strong consistency for ACL updates vs. eventual consistency.** Full synchronous re-indexing on every privacy change would slow the write path; instead, near-real-time async propagation (seconds-level lag) is accepted for ranking/content updates, while the *final read-time authorization check* is done against the live graph, so even if the cached index metadata is briefly stale, the last-mile check still prevents leaks — availability/freshness is relaxed but the actual security boundary is not.
- **Gateway-only JWT validation vs. per-service re-validation.** Validating only at the gateway is faster but makes the gateway a single trust anchor; the design compensates by having internal calls re-authenticate via mTLS service identity, so a compromised internal service still can't impersonate an arbitrary user without also holding a valid mTLS identity.

### How Real Systems Solve This

Meta's **Unicorn**, described in the VLDB paper *"Unicorn: A System for Searching the Social Graph"* (Curtiss et al., 2013), is the real precedent for this design: it models Facebook's data (including posts) as a graph and builds an inverted-index-like structure where each posting list can be intersected with graph-derived sets (e.g., "friends of user X") *at query time*, which is exactly the live-graph-intersection idea used above for `FRIENDS`/`FRIENDS_OF_FRIENDS` audiences rather than baking group membership into the document. Unicorn explicitly treats privacy/visibility as a query-time set intersection against the social graph rather than a static, pre-computed filter, which is the core lesson this case study borrows.

Mapping to OWASP API Security Top 10: this design's central concern is preventing **API1 (Broken Object Level Authorization)** — the single biggest risk in a search API, where an ID or index entry alone must never imply visibility. It also addresses **API3 (Broken Object Property Level Authorization)** by ensuring the search response only ever returns fields the viewer is entitled to (e.g., not leaking a private post's existence even in aggregate counts), **API4 (Unrestricted Resource Consumption)** via rate limiting and pagination caps at the gateway, and **API8 (Security Misconfiguration)** by centralizing TLS/mTLS and JWT validation configuration rather than re-implementing it ad hoc per service.

## Sources

- [RFC 7636: Proof Key for Code Exchange by OAuth Public Clients](https://www.rfc-editor.org/rfc/rfc7636.html)
- [Authorization Code Flow with PKCE — Auth0 Docs](https://auth0.com/docs/get-started/authentication-and-authorization-flow/authorization-code-flow-with-pkce)
- [PKCE — OAuth 2.0 (oauth.net)](https://oauth.net/2/pkce/)
- [OWASP API Security Top 10 (2023 edition)](https://owasp.org/API-Security/editions/2023/en/0x00-header/)
- [OWASP API Security Top 10 2023 has been released — OWASP Foundation](https://owasp.org/blog/2023/07/03/owasp-api-top10-2023)
- [AWS Secrets Engine — Vault, HashiCorp Developer Docs](https://developer.hashicorp.com/vault/docs/secrets/aws)
- [Unicorn: A System for Searching the Social Graph — Meta Research](https://research.facebook.com/publications/unicorn-a-system-for-searching-the-social-graph/)
- [Unicorn: A System for Searching the Social Graph — VLDB PDF](https://www.vldb.org/pvldb/vol6/p1150-curtiss.pdf)
- [The Making of Facebook's Graph Search — IEEE Spectrum](https://spectrum.ieee.org/the-making-of-facebooks-graph-search)
