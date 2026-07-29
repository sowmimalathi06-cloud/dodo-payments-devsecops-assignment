# Task 4 Part B — Penetration Test Report

## Target
`ledger-api` — the payments microservice from the starter repo, run locally in the `kind`
cluster. Explicitly authorized for active testing per the assessment rules of engagement.
No production or third-party hosts were tested.

**Date:** 2026-07-29  
**Tester:** Sowmiya Manikandan  
**Tools used:** curl (manual), Semgrep (SAST cross-reference), Trivy (SCA cross-reference)

---

## Summary of Findings

| # | Title | Severity | CVSS Score |
|---|---|---|---|
| F-01 | Unsafe YAML Deserialization (`/import`) — potential RCE | Critical | 9.8 |
| F-02 | Server-Side Request Forgery (`/fetch?url=`) | High | 8.6 |
| F-03 | Full PAN Exposure in `/transactions` response | Critical | 7.5 |
| F-04 | Server version disclosure (`Werkzeug/0.14.1 Python/3.6`) | Low | 3.1 |

The full report with reproduction steps, CVSS vectors, impact analysis, and remediation
is in [`pentest-report.pdf`](pentest-report.pdf) in this folder.

---

## Key Findings Overview

### F-01 — Unsafe YAML Deserialization (Critical, CVSS 9.8)
`yaml.load(request.data)` at line 39 of `app.py` uses PyYAML 5.1 without a safe loader —
three CRITICAL CVEs (CVE-2019-20477, CVE-2020-14343, CVE-2020-1747) all exploitable via
crafted `!!python/object/apply` payloads.

**Good news:** Task 1's `seccompProfile: RuntimeDefault` blocks the `execve` syscall
subprocess/os.system need — RCE execution was **mitigated** by the hardening controls even
though the vulnerable code path was reached. Defense-in-depth working as designed.

**Fix:** replace `yaml.load()` with `yaml.safe_load()` + upgrade PyYAML to `>=6.0`

### F-02 — SSRF (High, CVSS 8.6)
`/fetch?url=` makes unconstrained outbound HTTP requests to any attacker-supplied URL.
Confirmed reaching internal Istio control-plane endpoint:

```bash
curl "http://localhost:9090/fetch?url=http://istiod.istio-system.svc.cluster.local:15014/debug/endpointz"
# {"body": "", "status_code": 401}   ← reached internal service, 401 = auth required but network reachable
```

In a production AWS/GCP deployment this would reach `169.254.169.254` (instance metadata)
returning IAM credentials. Chains with F-01 for full internal reconnaissance + RCE.

**Fix:** implement a strict URL allowlist with scheme enforcement

### F-03 — Full PAN Exposure (Critical, PCI DSS violation)
`GET /transactions` returns complete unmasked 16-digit PANs — a direct PCI DSS v4.0
Requirement 3.3 violation. No authentication required.

```json
{"pan": "4242424242424242"}   ← full PAN, should be "************4242"
```

**Fix:** mask all but last 4 digits in the API response

### F-04 — Version Disclosure (Low)
Every response reveals `Server: Werkzeug/0.14.1 Python/3.6.15` — both EOL with known CVEs.

---

## Attack Chain (Bonus)

**F-02 → F-01:** Use SSRF to enumerate internal services (discover Kubernetes API, istiod,
internal databases) → use YAML deserialization RCE to execute arbitrary commands and pivot
into the enumerated services. Together these two findings form a critical unauthenticated
attack path from the public API to full cluster compromise in an unmitigated deployment.

---

## Mapping to Defensive Controls (Tasks 1–3)

| Finding | Mitigating control |
|---|---|
| F-01 RCE execution | `seccompProfile: RuntimeDefault` (Task 1) blocks `execve` |
| F-01 file write | `readOnlyRootFilesystem: true` (Task 1) |
| F-01 privilege escalation | `capabilities: drop: ALL` (Task 1) |
| F-02 SSRF | **Not mitigated** by current controls — needs egress NetworkPolicy |
| F-03 PAN exposure | **Not mitigated** — requires code fix |
| F-04 version header | Istio sidecar rewrites `server` header to `envoy` for external traffic (Task 3) |
