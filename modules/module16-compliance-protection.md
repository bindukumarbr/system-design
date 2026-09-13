# Module 16: Compliance & Protection

## Core Concepts

### Web Application Firewall (WAF)

A WAF sits between the internet and your application (as a reverse proxy, CDN edge module, or load-balancer plugin) and inspects HTTP requests/responses against a rule set before they reach application code. It operates at L7, unlike a network firewall (L3/L4).

Rule categories:
- **Managed rulesets** for known attack signatures — SQLi, XSS, path traversal, RCE payloads, protocol anomalies (Cloudflare Managed Ruleset, AWS Managed Rules, ModSecurity CRS).
- **Custom rules** — block/allow by header, path, geography, ASN, JA3/JA4 TLS fingerprint.
- **Rate-based rules** — count requests per key (IP, session, API key) over a rolling window and challenge/block on threshold.
- **Bot management** — heuristic + ML scoring of "bot likelihood" using TLS fingerprinting, behavioral signals (mouse movement, navigation timing), and IP reputation.

Two deployment modes matter architecturally: **positive security model** (allow-list known-good request shapes — schema validation for APIs) is far stronger than a **negative model** (block known-bad signatures) but requires you to maintain the allow-list as the API evolves. For a public API, pairing an OpenAPI-schema-validating WAF rule with the negative-model managed ruleset gives defense in depth. AWS WAF and Cloudflare both support "block on schema violation" plus managed threat-intel rule groups (Cloudflare, 2026).

Where it sits in the request path: DNS → Anycast edge (WAF + DDoS scrubbing) → CDN/cache → rate limiter → load balancer → app. Terminating TLS at the edge lets the WAF see plaintext, which is why cloud WAFs are usually bundled with the CDN/edge layer.

### Rate Limiting and Abuse Prevention

Rate limiting protects capacity and fairness; abuse prevention protects business logic. They need different mechanisms:

- **Fixed/sliding window counters** (Redis `INCR` + TTL, or sliding-window log) for simple per-key throttling.
- **Token bucket / leaky bucket** for smoothing bursts while enforcing an average rate — the standard choice for public APIs because it allows short bursts without needing per-second precision.
- **Layered limits**: per-IP (cheap, easily evaded via botnets/residential proxies), per-API-key/user (harder to evade, ties to billing tier), per-endpoint (protect expensive endpoints like search or export separately from cheap reads).
- **Bot detection** beyond rate counting: TLS/JA4 fingerprint anomalies, missing/inconsistent headers, headless-browser tells (WebDriver flags, canvas/WebGL fingerprint entropy), IP reputation/ASN (datacenter vs residential), and behavioral scoring. Cloudflare's Bot Management assigns a 1–99 bot score combining these signals and lets you branch on score (allow/managed challenge/JS challenge/block) rather than a binary decision (Cloudflare Bot Solutions docs, 2026).
- **Scraping mitigation specifically**: CAPTCHA/JS challenges raise the cost of automation but hurt legitimate accessibility; better is a graduated response — degrade (add latency, serve cached/stale data, reduce result density) before hard-blocking, since outright blocking a scraper's IP just causes rotation. Honeypot fields, canary tokens embedded in HTML, and anomaly detection on request cadence (too-regular intervals) catch naive scrapers cheaply.
- Return `429 Too Many Requests` with `Retry-After`, and expose `X-RateLimit-Limit/Remaining/Reset` so well-behaved clients self-throttle — this is also what you want *your own* outbound scrapers to respect on target sites.

### RBAC vs ABAC

**RBAC (Role-Based Access Control)**: permissions attach to roles (admin, editor, viewer); users are assigned roles. Simple to reason about, easy to audit ("who has admin"), cheap to implement (a roles table + a permission-check middleware). Weakness: role explosion — as conditions multiply ("editor, but only for their own team, only during business hours, only for records under $10k") you either create combinatorial roles (`editor_team_a_business_hours`) or bolt ad hoc conditionals onto the RBAC check, defeating its simplicity.

**ABAC (Attribute-Based Access Control)**: a policy engine evaluates rules against attributes of the subject (user, role, department), resource (owner, sensitivity tag, region), action, and environment (time, IP, device posture). Expressed declaratively, e.g., `allow(read, document) if user.department == document.department AND user.clearance >= document.sensitivity`. NIST SP 800-162 formalizes this model. ABAC scales to fine-grained, context-dependent authorization without a combinatorial explosion of roles, and centralizes policy (good for audit/compliance), but costs you: a policy engine (OPA/Cedar/custom), harder-to-audit "why was this allowed" questions since policies are compositional, and higher latency per authorization decision unless cached.

**When ABAC is worth it**: multi-tenant B2B platforms with partner-tier APIs, data with regulatory sensitivity tags (PHI/PII) requiring purpose-based access, or systems where "who can see what" depends on relationships (ownership, org hierarchy) rather than a fixed role. **When RBAC suffices**: internal tools, small permission surface, few roles that rarely combine conditions. A common pragmatic middle ground is RBAC-for-coarse-gating + ABAC-for-resource-scoping — role determines *what actions exist*, attributes determine *which resource instances* they apply to (Oso, Wiz, Okta comparisons, 2026).

### OWASP Top 10 → Architectural Decisions

| OWASP risk | Architectural response |
|---|---|
| A01 Broken Access Control | Deny-by-default authorization middleware; enforce checks server-side per resource (not just per route); use ABAC/policy engine for resource-level checks; disable directory listing |
| A02 Cryptographic Failures | TLS everywhere including internal service-to-service; encrypt PII at rest (KMS-managed keys); never roll your own crypto; short-lived tokens |
| A03 Injection | Parameterized queries/ORMs by default; input validation at the API gateway (schema validation ties back to WAF positive-security model); least-privilege DB accounts |
| A04 Insecure Design | Threat model at design time; rate limits and abuse cases as first-class NFRs, not bolted on later |
| A05 Security Misconfiguration | Infrastructure as code with security baselines; automated config scanning (CIS benchmarks); no default credentials |
| A06 Vulnerable and Outdated Components | SBOM + automated dependency scanning in CI (see below) |
| A07 Identification and Authentication Failures | MFA, short session lifetimes, rate-limited login endpoints, breached-password checks |
| A08 Software and Data Integrity Failures | Signed artifacts/images, CI/CD pipeline integrity (SLSA provenance), verify third-party package checksums |
| A09 Security Logging and Monitoring Failures | Centralized audit logs for authz decisions, anomaly alerting, immutable log storage for forensics |
| A10 Server-Side Request Forgery (SSRF) | Egress allow-lists for outbound calls, block requests to internal IP ranges from services that fetch user-supplied URLs — directly relevant to a scraping service fetching arbitrary product URLs |

(OWASP Top 10 2021; Google Cloud's OWASP mitigation mapping, 2026.)

### Data Residency and Compliance Regimes

- **GDPR** (EU): applies whenever you process EU residents' personal data, regardless of where your company is based. Architecturally requires: a lawful basis + consent management, data subject rights (export/delete on request — needs a data model that can locate *all* records for a user across services), breach notification within 72 hours (requires detection/logging), and often data residency — many customers/regulators expect EU personal data to stay in EU data centers, which drives multi-region deployments with region-pinned storage and routing.
- **SOC 2** (US, attestation not law): Trust Services Criteria (security, availability, processing integrity, confidentiality, privacy). Architecturally this is less about *where* data lives and more about *provable controls*: access reviews, change management, encryption, incident response, vendor risk management, and continuous logging/monitoring evidence for auditors. It's an operational-maturity attestation your enterprise customers require before they'll sign a contract.
- **HIPAA** (US healthcare): governs PHI. Requires signed Business Associate Agreements with every subprocessor touching PHI, encryption at rest and in transit, strict audit logging of every access to PHI (who/when/what), minimum-necessary access (pushes toward ABAC), and often physical/logical isolation of PHI-handling infrastructure.

Common thread architecturally: all three push you toward (1) knowing precisely where each class of data lives (data classification + inventory), (2) fine-grained, logged access control, (3) encryption as a baseline, and (4) the ability to prove controls via logs/audits — not just implement them once.

### Supply Chain Security (SBOM, Dependency Scanning)

An **SBOM (Software Bill of Materials)** is a machine-readable inventory of every component (direct and transitive) in a build, typically in SPDX or CycloneDX format. **Syft** generates SBOMs from container images or filesystems; pairing it with **Grype** scans that SBOM against vulnerability databases (OWASP Dependency-Graph/SBOM Cheat Sheet, 2026). **Dependabot** (GitHub-native) opens PRs to bump vulnerable dependencies and can also generate a dependency graph SBOM automatically. **Snyk** does dependency + container + IaC scanning with a hosted vulnerability DB and policy gates in CI.

Architectural practice: generate an SBOM at build time (not after the fact), attach it to the release artifact, scan it in CI as a merge-blocking gate for critical/high CVEs, and re-scan continuously (new CVEs are disclosed after your last build). For higher assurance, add SLSA provenance attestation so consumers can verify what pipeline produced an artifact and that it wasn't tampered with post-build.

---

## Case Study Solution: Price Tracking Service

### Problem Statement & Clarifying Requirements

Build a service that tracks prices of products across multiple e-commerce sites over time and alerts users when a tracked product's price drops below a threshold.

**Functional requirements**
- Users add a product URL (or search) to track; system periodically fetches its current price.
- Store full price history per product.
- Users set an alert threshold ("notify me if price < $X" or "any drop ≥ 10%").
- Public API for querying price history and managing subscriptions; a paid "partner tier" gets higher-rate programmatic access.

**Non-functional requirements**
- Must scrape target sites *responsibly*: respect their rate limits/robots.txt, back off on 429s/blocks, avoid tripping their anti-bot defenses in a way that gets the service IP-banned.
- Must defend its *own* public API from abuse (scraping-of-the-scraper, credential stuffing, unbounded polling).
- Multi-region (EU users) → data residency considerations.
- High read volume (price history reads) vs. moderate write volume (periodic scrape ingestion).
- Alert delivery latency: minutes, not seconds, is acceptable.

### Capacity Estimation

Assume 5M tracked products, average re-check interval 4 hours (6 checks/day/product):
- Scrape writes: 5M × 6 = 30M price-checks/day ≈ 350/sec average, bursty around scheduling windows.
- Price history storage: 30M rows/day × ~150 bytes/row ≈ 4.5 GB/day, ~1.6 TB/year uncompressed — a time-series-friendly store with columnar compression (e.g., Timescale/Redshift-style) gets this down 5–10x.
- Alerts: assume 2% of checks trigger a threshold ⇒ 600K alert evaluations/day, of which maybe 5% actually notify → ~30K notifications/day, trivial volume for a queue-backed notifier.
- Public API read traffic: assume 500K DAU checking dashboards a few times/day → ~5M API reads/day ≈ 60 req/sec average, with peak multipliers (3–5x) for diurnal patterns — the number the rate limiter and cache must be sized around.

### High-Level Architecture

```
                       ┌─────────────────────────┐
 Users / Partner API   │   Edge: WAF + Bot Mgmt   │
        │              │  + Rate Limiter (per key)│
        ▼              └────────────┬────────────┘
 ┌─────────────┐                    ▼
 │   Public    │        ┌───────────────────────┐
 │   API GW    │◄──────►│  Public API Service    │
 └─────────────┘        │ (RBAC coarse + ABAC    │
                         │  resource-scoped authz)│
                         └───────┬───────┬────────┘
                                 │       │
                    ┌────────────┘       └────────────┐
                    ▼                                 ▼
          ┌──────────────────┐              ┌──────────────────┐
          │ Price History DB │              │ Subscriptions DB  │
          │ (time-series)    │              │ (Postgres)        │
          └────────▲─────────┘              └─────────▲─────────┘
                   │                                    │
          ┌────────┴─────────┐                ┌─────────┴────────┐
          │ Ingestion Workers │───publish────►│  Alert Evaluator  │
          │ (per-site adapter,│   price update │  + Notifier      │
          │  backoff, proxy   │                │  (queue-driven)  │
          │  pool, robots.txt │                └─────────┬────────┘
          │  compliance)      │                          ▼
          └────────┬──────────┘                 Email/Push/Webhook
                   │
                   ▼
          Scrape Job Scheduler
          (rate-limited per-target-site,
           exponential backoff on 429/403)
                   │
                   ▼
           Third-party e-commerce sites
```

Key flow decisions:
- **Outbound respect**: the scheduler enforces a *per-target-domain* token bucket (not just per-worker), independent from inbound rate limiting, so 50 parallel workers don't collectively hammer one site. Backoff is exponential with jitter on 429/403/CAPTCHA-detected responses, and a circuit breaker pauses a domain entirely if block rate spikes.
- **Inbound protection**: WAF + bot management + rate limiter sit in front of the API gateway, before any app logic runs — cheap requests (cached reads) are absorbed at the CDN edge; expensive requests (raw price-history export, bulk queries) get their own stricter per-endpoint limit.
- Ingestion and the public API are decoupled via the databases/queue so a scraping slowdown never degrades API read latency.

### API Design

```
GET  /v1/products/{product_id}/price-history?from=&to=&granularity=day
     → paginated time series [{timestamp, price, currency, source_url}]

POST /v1/subscriptions
     body: { product_id, threshold_type: "absolute"|"percent_drop", value }
     → creates an alert subscription for the caller

GET  /v1/subscriptions
     → list caller's active subscriptions

DELETE /v1/subscriptions/{id}

# Partner tier (higher rate limit, API-key auth)
GET  /v1/partner/products/{product_id}/price-history/bulk
POST /v1/partner/products/batch-lookup
```

All endpoints require an API key (partner tier) or session token (consumer tier); rate-limit tier is resolved from the key/token at the gateway before hitting the app.

### Data Model

```sql
-- Product catalog
products(
  id UUID PK, canonical_url TEXT, title TEXT, retailer TEXT,
  region TEXT,           -- data residency tag: 'EU' | 'US' | 'GLOBAL'
  created_at TIMESTAMPTZ
)

-- Price history (time-series, partitioned by day/product)
price_history(
  product_id UUID FK, ts TIMESTAMPTZ, price NUMERIC, currency CHAR(3),
  source_snapshot_url TEXT,
  PRIMARY KEY (product_id, ts)
) PARTITION BY RANGE (ts)

-- User alert subscriptions
subscriptions(
  id UUID PK, user_id UUID FK, product_id UUID FK,
  threshold_type TEXT, threshold_value NUMERIC,
  region TEXT,           -- pinned to user's home region for residency
  created_at TIMESTAMPTZ
)

-- Access control metadata
api_keys(
  id UUID PK, owner_id UUID, tier TEXT,          -- 'consumer'|'partner'|'admin'
  rate_limit_override INT, region_scope TEXT,
  created_at TIMESTAMPTZ, revoked_at TIMESTAMPTZ
)
policies(  -- ABAC rules, evaluated for resource-level checks
  id UUID PK, effect TEXT, action TEXT,
  subject_attr JSONB, resource_attr JSONB, condition JSONB
)
```

### Deep Dive

**RBAC vs ABAC for this system.** Three coarse actor classes exist — consumer user, admin, partner API — which maps cleanly onto RBAC for the *first cut* of authorization ("can this caller hit this endpoint at all"). But resource-level questions don't fit roles well: "can partner X read price history for product Y" depends on the partner's contracted region scope, whether product Y is region-restricted (EU-only retailer data with contractual redistribution limits), and whether the *user's own* subscriptions are visible only to that user. That's exactly the shape ABAC handles: role gates the action space, attribute policies (`policies` table above) gate the resource instance. This hybrid avoids partner-specific role explosion (`partner_tier1_eu`, `partner_tier2_global`, …) while keeping the common case (regular user CRUD on their own subscriptions) cheap — a simple ownership check, not a full policy evaluation.

**WAF and rate limiting placement.** Both sit at the edge, ahead of the API gateway and app tier, as shown in the diagram. Bot management scores requests before they even reach rate-limit counters — a request scored as "likely automated" gets a stricter bucket or a challenge, while browser-verified traffic gets the normal consumer limit. Per-endpoint limits matter specifically here: `/price-history` (cheap, cacheable) can tolerate a generous limit; `/partner/batch-lookup` (expensive, DB-heavy) gets a tight limit tied to the partner's contracted quota, enforced both at the edge (coarse) and in the app (precise, since partner billing depends on accurate counts).

**Data residency.** EU user subscriptions and any personal data (email, alert preferences) are tagged `region = 'EU'` and physically stored in an EU region; the price-history time series itself (facts about public product prices, not personal data) doesn't require residency but is still partitioned regionally for latency. Cross-region joins (e.g., an EU user tracking a US retailer's product) are handled by keeping the *personal* subscription row in-region while referencing a globally-replicated read-only product/price-history dataset — this satisfies GDPR's data-residency expectation without forcing full data duplication of the price catalog.

**Supply-chain security for ingestion workers.** Ingestion workers pull HTML/JSON from dozens of third-party sites and often depend on a wide surface of scraping/parsing libraries (HTTP clients, HTML parsers, proxy-rotation SDKs) — a high-value target for dependency compromise. Practice: generate an SBOM (Syft) for every worker container image at build time, scan it (Grype/Snyk) as a CI gate blocking on critical/high CVEs, and run Dependabot for continuous PR-based patching between builds. Because workers execute untrusted response content (parsing arbitrary third-party HTML), sandbox the parsing step (no shell-out, no `eval`, memory/time limits per parse) — this is an OWASP A08 (software/data integrity) and general untrusted-input concern layered on top of the supply-chain hygiene.

### Trade-offs and Alternatives Considered

- **Polling interval (4h) vs. webhook/push from retailers**: retailers don't offer this, so polling is forced; the trade-off is between freshness (shorter interval → faster alerts) and ban risk/cost (shorter interval → more requests → higher chance of triggering the target's own anti-bot defenses). Mitigate by adaptive intervals: poll fast-moving high-traffic products more often, slow-moving ones less.
- **RBAC-only (simpler) vs. RBAC+ABAC hybrid (chosen)**: RBAC-only would be cheaper to build but forces role explosion once partner-tier region scoping is added; the hybrid costs an extra policy-evaluation hop but avoids that explosion.
- **Single global DB vs. region-sharded**: a single global Postgres is simpler operationally but fails GDPR residency expectations for EU personal data; region-sharding subscriptions (chosen) adds cross-region query complexity but satisfies residency without full geo-replication of the whole dataset.
- **Hard-block vs. graduated response to suspected scraping-of-us**: hard-blocking is simpler but false-positives hurt legitimate power users; a graduated response (cache/stale-serve, then challenge, then block) is more engineering effort but better preserves legitimate traffic.

### How Real Systems Solve This

CamelCamelCamel and Keepa (Amazon price trackers) rely heavily on cached/throttled polling against Amazon's own product APIs/pages and are well known for aggressive backoff and IP-rotation discipline to avoid being blocked — illustrating exactly the outbound-politeness problem this design addresses. On the inbound-abuse side, general industry practice (Cloudflare's Bot Management, AWS WAF Bot Control) is what most SaaS APIs — including price-comparison and deal-alert platforms — front their public endpoints with, using tiered rate limits keyed to API plan rather than IP alone, since scraper traffic routinely originates from residential proxy pools that defeat pure IP-based limiting.

---

## Sources

- [OWASP Top 10 2021 mitigation options on Google Cloud](https://cloud.google.com/architecture/security/owasp-top-ten-mitigation)
- [OWASP Top 10 Cheat Sheet: Threats and Mitigations in Brief](https://www.pynt.io/learning-hub/owasp-top-10-guide/owasp-top-10-cheat-sheet-threats-and-mitigations-in-brief)
- [RBAC vs ABAC: main differences and which one you should use — Oso](https://www.osohq.com/learn/rbac-vs-abac)
- [ABAC vs. RBAC: What's The Difference? — Wiz](https://www.wiz.io/academy/cloud-security/abac-vs-rbac)
- [RBAC vs. ABAC: Definitions & When to Use — Okta](https://www.okta.com/identity-101/role-based-access-control-vs-attribute-based-access-control/)
- [Dependency Graph / SBOM Cheat Sheet — OWASP](https://cheatsheetseries.owasp.org/cheatsheets/Dependency_Graph_SBOM_Cheat_Sheet.html)
- [Dependabot vs Snyk vs Trivy vs npm audit: SCA tool comparison](https://tomodahinata.com/en/blog/dependabot-vs-snyk-trivy-npm-audit-sca-tools-comparison-guide)
- [Rate limiting best practices — Cloudflare WAF docs](https://developers.cloudflare.com/waf/rate-limiting-rules/best-practices/)
- [Cloudflare Bot Management](https://www.cloudflare.com/products/bot-mitigation/)
- [Cloudflare Bot Solutions overview](https://developers.cloudflare.com/bots/)
- [From GDPR to SOC 2: A Practical Guide to Building Compliance into Your Software](https://medium.com/@aleyacyrus/from-gdpr-to-soc-2-a-practical-guide-to-building-compliance-into-your-software-7416422ba374)
- [SOC 2 vs GDPR Explained: Key Differences, Overlaps, and Smart Compliance Mapping — Sprinto](https://sprinto.com/blog/soc-2-vs-gdpr/)
