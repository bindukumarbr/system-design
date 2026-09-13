# Fast-Track: Module 15 - Security Architecture
**Core Concept:** Zero Trust, OAuth 2.0/OIDC/PKCE, JWT at Edge, mTLS, OWASP API Top 10.
**Case Study:** Facebook Post Search
- **Key Insight:** Prevent BOLA (Broken Object Level Authorization) by evaluating ACLs dynamically at query time for every object, not just routing.
- **Takeaway:** Terminate/validate JWTs at the API Gateway. Use mTLS for internal service identity.
