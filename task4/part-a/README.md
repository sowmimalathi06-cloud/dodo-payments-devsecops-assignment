# Task 4 Part A — Passive Reconnaissance Report: dodopayments.tech

**Scope:** Passive OSINT only — DNS, certificate transparency logs, and standard HTTP probing.
No active scanning, fuzzing, or exploitation was performed against any `dodopayments.tech`
or `dodopayments.com` host, per the rules of engagement.

**Date:** 2026-07-28
**Tools used:** subfinder, assetfinder, crt.sh, httpx, testssl.sh

---

## 1. Methodology

1. Subdomain enumeration via `subfinder` (aggregates multiple OSINT sources including
   certificate transparency) and `assetfinder`.
2. Deduplicated and merged into a single subdomain list — **135 unique names** discovered
   (11 were wildcard certificate entries, filtered out before probing).
3. Live-host fingerprinting via `httpx` against the remaining **124 hosts** — capturing HTTP
   status code, page title, detected technologies, and server/CDN.
4. TLS/SSL posture check via `testssl.sh` against the main domain and one exposed internal
   tool, to sample the edge TLS configuration.

---

## 2. Subdomain Enumeration Summary

**135 subdomains** discovered via certificate transparency logs and passive DNS aggregation.
Of these, wildcard entries (`*.app.`, `*.customer.`, `*.store.`, etc.) indicate several
subdomains are served dynamically/multi-tenant (likely per-customer or per-environment
routing), which is common for a SaaS billing platform.

Notable naming patterns observed:
- **Environment staging:** `dev.`, `test.`, `testing.`, `demo.`, `staging`-style prefixes
  appear repeatedly (`clickhouse-dev`, `clickhouse-dev-v2`, `events-dev`, `sequindev`,
  `svix-dev`, `signoz-dev`) alongside their `-prod`/`-v2` counterparts — suggesting active
  dev/prod parity that could leak non-production data or weaker controls if misconfigured.
- **Internal tooling namespace:** a dedicated `*.infra.dodopayments.tech` naming convention
  groups internal infrastructure tools (`sentry-prod.infra`, `signoz-prod.infra`,
  `svix-dev.infra`, `kafka-ui-*.infra`, `firecrawl.infra`, `usesend.infra`, `prowler.infra`).

---

## 3. Live Host Inventory & Risk Observations

Of 124 probed hosts, roughly 50 responded with a live HTTP status. The table below groups
the most security-relevant findings — full raw output is in `httpx-results.txt`.

### 3.1 Exposed internal/admin tooling (HIGH interest to an attacker)

| Host | Status | Technology | Why it matters |
|---|---|---|---|
| `sonarqube.dodopayments.tech` | 200 | SonarQube, Java | Static code analysis platform — if accessible, can leak source code structure, quality findings, and sometimes credentials/secrets flagged in scan history. High-value target for source code disclosure. |
| `keycloak.dodopayments.tech` | 200 (login) | Keycloak Admin Console | Identity/SSO provider admin console reachable. If admin login is exposed with weak/default creds, this is a full identity-provider compromise path — realistically the highest-impact single finding in this list. |
| `n8n.dodopayments.tech` | 200 | n8n.io Workflow Automation | Workflow automation tool — often holds API keys/credentials for connected services (Slack, email, databases) in its credential store. |
| `mb.dodopayments.tech` | 200 | Metabase | BI/analytics dashboard — potential access to business/customer data if not properly authenticated. |
| `clickhouse-prod-v2.dodopayments.tech` | 200 | ClickHouse | Production analytics database HTTP interface reachable directly. ClickHouse's default HTTP interface has a history of unauthenticated query execution when misconfigured — worth a closer (in-scope, authorized) look. |
| `clickhouse-dev-v2.dodopayments.tech` | 200 | ClickHouse | Dev counterpart to the above — same concern, plus dev environments often have looser auth. |
| `codecov.dodopayments.tech` | 200 | Codecov | Code coverage reporting tied to CI — can hint at repository structure and test coverage gaps. |
| `sentinal-v2.dodopayments.tech` | 200 (login via Vercel) | Next.js/React, Vercel | Internally-named "Sentinel" app — naming suggests a security/monitoring tool; worth noting as unknown internal app exposed publicly. |

### 3.2 Cloudflare-protected / access-controlled (lower immediate risk, still noted)

Several sensitive-looking hosts return **403 Forbidden** or **401 Unauthorized** directly
from Cloudflare, indicating either a Cloudflare Access policy or WAF rule is already blocking
public access:

`internal.dodopayments.tech` (403), `sentry.dodopayments.tech` (403), `test.dodopayments.tech`
(403), `live.dodopayments.tech` (403), `ai-proxy.dodopayments.tech` (401),
`ssr-proxy.dodopayments.tech` (401)

**Observation:** this is a good sign — it shows some internal hosts are deliberately gated.
However, the subdomain names themselves are still disclosed via certificate transparency
regardless of the access control, which is useful reconnaissance for an attacker even
without direct access (confirms the existence of internal Sentry, an "ai-proxy", etc.).

### 3.3 Customer/product-facing surface (expected, standard due diligence)

`app.dodopayments.tech`, `checkout.dodopayments.tech`, `store.dodopayments.tech`,
`customer.dodopayments.tech`, `partners.dodopayments.tech`, `dodopayments.tech` (marketing site)
— these are the expected public product surface. Most sit behind Vercel with a security
checkpoint (429 "Vercel Security Checkpoint" — likely bot/rate-limit protection) or standard
Next.js apps behind Cloudflare. No immediate concerns from passive probing alone; these are
the natural Part B pentest candidates if in scope.

### 3.4 Raw GCP IPs indexed as subdomains

`146.148.dodopayments.tech`, `34.29.dodopayments.tech`, `35.244.dodopayments.tech` — these
look like GCP ephemeral IPs that were issued a certificate under the `dodopayments.tech`
zone (possibly via wildcard cert or automated cert issuance for load balancers). Worth
flagging: automated/wildcard certs can inadvertently create certificate-transparency
log entries for infrastructure that wasn't meant to be discoverable this way — a known
CT-log OSINT trade-off.

---

## 4. TLS/SSL Posture

Sampled `dodopayments.tech` (marketing site) and `sonarqube.dodopayments.tech`:

| Finding | Detail |
|---|---|
| Overall Grade | **B** (SSL Labs-style rating via testssl.sh) |
| Grade cap reason | TLS 1.0 and TLS 1.1 still offered alongside TLS 1.2/1.3 |
| Strong protocol support otherwise | TLS 1.3 with modern cipher suites (AES-GCM, ChaCha20-Poly1305) and X25519/X25519MLKEM768 key exchange available for modern clients |
| TLS termination | Both sampled hosts terminate TLS at **Cloudflare's edge** — the grade reflects Cloudflare's TLS config, not Dodo's origin servers directly |

**Recommendation:** disable TLS 1.0/1.1 at the Cloudflare edge (SSL/TLS → Edge Certificates →
Minimum TLS Version) to reach an A grade and remove legacy-protocol exposure, which offers
no practical benefit given all sampled clients support TLS 1.2+.

---

## 5. Summary — What a real attacker would find interesting

1. **Keycloak admin console** publicly reachable is the single highest-value target — an
   attacker would immediately attempt credential stuffing, default credentials, or known
   Keycloak CVEs against this before anything else.
2. **SonarQube and Codecov** exposure gives an attacker insight into the codebase without
   needing repo access — potential for leaked secrets in scan history or coverage reports.
3. **Dev/prod naming parity** (`clickhouse-dev-v2` next to `clickhouse-prod-v2`, etc.)
   suggests testing environment misconfiguration risk — dev environments are worth checking
   for weaker auth than their prod counterparts.
4. **n8n and Metabase** exposure represent potential credential/data leakage points if
   authentication is weak, since both tools commonly store third-party API credentials or
   surface internal business data.
5. Overall attack surface is **large** (135 subdomains) for what is fundamentally a payments
   company — most of it appears to be internal tooling that ended up publicly resolvable via
   DNS/certificates even where access itself is gated by Cloudflare. Reducing the *discoverable*
   surface (e.g., issuing internal-tool certs off a separate, non-publicly-logged CA/zone,
   or using split-horizon DNS) would meaningfully shrink what shows up in this kind of recon,
   independent of the access controls already in place.

---

## Appendix: Raw data files
- `subfinder-results.txt` — subfinder output
- `assetfinder-results.txt` — assetfinder output
- `all-subdomains.txt` — merged, deduplicated list (135 entries)
- `httpx-results.txt` — full live-host probe results
- `tls-checks/main-domain.txt` — full testssl.sh output for dodopayments.tech
- `tls-checks/sonarqube.txt` — full testssl.sh output for sonarqube.dodopayments.tech
