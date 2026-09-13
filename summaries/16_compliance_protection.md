# Module 16: Compliance & Protection

## Core Concepts

### Web Application Firewall (WAF) & Rate Limiting
- **WAF:** Operates at L7. Blocks SQLi, XSS, malicious bot signatures. 
  - *Positive security model:* Only allow JSON that exactly matches an OpenAPI schema.
  - *Negative security model:* Block known bad signatures.
- **Rate Limiting:** Protects capacity.
  - **Token Bucket Algorithm:** Used for APIs. Allows short bursts but enforces a sustained average rate.
  - Rate limiting is applied by IP, API key, and even per-endpoint (e.g., cheap reads vs expensive exports).
- **Bot Management:** Uses TLS fingerprints, JavaScript challenges, and behavioral scoring to block scrapers.

### RBAC vs ABAC
- **RBAC (Role-Based Access Control):** Users have roles (Admin, Editor). Simple, but leads to "role explosion" when rules get complex (`editor_team_a_weekend_only`).
- **ABAC (Attribute-Based Access Control):** Uses declarative rules (`allow if user.dept == doc.dept`). Scales to fine-grained context but is slower and harder to audit. 
- *Best Practice:* Use RBAC for coarse gating ("can they hit this API?") and ABAC for resource scoping ("can they see this specific row?").

### Compliance Regimes
- **GDPR (EU):** Dictates user consent, data export, and "right to be forgotten". Drives Data Residency architecture (keeping EU data in EU datacenters).
- **SOC 2:** Attestation of operational maturity (audits, change management, encryption). Required for B2B enterprise sales.
- **HIPAA:** US healthcare. Strict PHI tracking, encryption, and audit logging.

### Supply Chain Security
- **SBOM (Software Bill of Materials):** A machine-readable list of every dependency in your build.
- **Dependency Scanning:** Running `Syft` and `Grype` (or Snyk/Dependabot) in your CI/CD pipeline to block merges if a critical CVE is found.

---

## Case Study: Price Tracking Service
- **Requirements:** Track product prices on e-commerce sites. Alert users on price drops. Provide an API for partners.
- **Estimations:** 30M scrapes/day. High read volume for public dashboards.
- **Architecture:** 
  - **Outbound Ingestion Workers:** Adhere to robots.txt, use exponential backoff on 429/403s, and use a token bucket per-target-domain so they don't DDOS target retailers.
  - **Inbound Public API:** Shielded by Edge WAF, Bot Management, and per-key rate limiting.
  - **Databases:** Subscriptions DB (Postgres, sharded by region for GDPR residency) + Price History DB (Time-series, columnar store).
- **Aha! Insights:**
  - **Data Residency Hack:** To comply with GDPR, EU users' personal *subscription* data is pinned to an EU Postgres instance. However, the *public price history* of the products is replicated globally. You don't have to duplicate the entire internet's price history, just the PII.
  - **Graduated Anti-Bot Response:** If the service suspects it is being scraped, don't hard block immediately (false positives hurt real users). Instead, degrade performance (add latency, serve cached data), then issue a JS challenge, and only hard block as a last resort.
