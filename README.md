# Dodo Payments — DevSecOps Assessment

**Candidate:** Sowmiya Manikandan  
**Role:** Security & DevOps Engineer  
**Repository:** https://github.com/sowmimalathi06-cloud/dodo-payments-devsecops-assignment

---

## Overview

This repository contains the complete submission for the Dodo Payments Security & DevOps
Engineer technical assessment. It covers end-to-end hardening of `ledger-api` — a
payments microservice handling cardholder-adjacent data — across four domains: workload
security, secure CI/CD supply chain, zero-trust service mesh networking, and offensive
security (reconnaissance + penetration testing).

Everything runs locally and free — a `kind` Kubernetes cluster (v1.30.0) + GitHub Actions
free-tier runners. No cloud account was used.

---

## Tasks

| Task | Status | Write-up |
|---|---|---|
| Task 1 — Deploy & Harden the Workload | ✅ Complete | [task1/README.md](task1/README.md) |
| Task 2 — Secure CI/CD Pipeline & Supply Chain | ✅ Complete | [task2/README.md](task2/README.md) |
| Task 3 — Service Mesh & Zero-Trust (Istio) | ✅ Complete | [task3/README.md](task3/README.md) |
| Task 4A — Passive Reconnaissance | ✅ Complete | [task4/part-a/README.md](task4/part-a/README.md) |
| Task 4B — Penetration Test | ✅ Complete | [task4/part-b/README.md](task4/part-b/README.md) |

---

## Highlights

### Task 1 — Workload Hardening
- Non-root container (`uid=10001`), read-only root filesystem, all capabilities dropped,
  `seccompProfile: RuntimeDefault` — all verified with `kubectl exec` evidence
- Secrets migrated from plaintext `deployment.yaml` to Bitnami **Sealed Secrets** (git-safe
  encrypted blob, plaintext never committed)
- Dedicated least-privilege `ServiceAccount` + scoped `Role` (read-only access to one
  named ConfigMap only) — verified with `kubectl auth can-i`
- **Kyverno** admission policies (`disallow-root-containers`, `disallow-latest-tag`,
  `require-signed-images`) — demonstrated live rejection of the original insecure Deployment

### Task 2 — Secure CI/CD
- 4-gate GitHub Actions pipeline: secrets scan (gitleaks) → SAST (Semgrep) → dependency
  CVE scan (Trivy) → image scan (Trivy) → push + Cosign keyless sign + SLSA attestation
- Pipeline **correctly blocked** the first run: Semgrep caught `yaml.load()` (RCE) and
  SSRF; Trivy found 3 CRITICAL CVEs in PyYAML 5.1 — the same root cause caught independently
  by two different tools (defense-in-depth)
- All scanner results uploaded as SARIF to GitHub Security tab
- **ArgoCD** GitOps with `selfHeal: true` — drift detection + self-heal demonstrated:
  manual `kubectl scale --replicas=1` auto-reverted to 3 replicas in ~7 seconds

### Task 3 — Zero-Trust Service Mesh
- Istio 1.22.5 installed (compatible with K8s 1.30; latest Istio requires 1.32+, noted)
- `PeerAuthentication: STRICT` — plaintext request from outside the mesh returns
  `Connection reset by peer` (curl exit 56), confirmed
- Default-deny `AuthorizationPolicy` + explicit allow keyed on SPIFFE/ServiceAccount
  identity — authorized caller (`reporting` SA) returns 200; unauthorized caller returns
  `403 RBAC: access denied` — proven live
- Kubernetes `NetworkPolicy` layered underneath for defense-in-depth (L3/L4 under L7)

### Task 4 — Offensive Security
- **Part A (Recon):** 135 subdomains discovered via CT logs/passive DNS; 50 live hosts
  fingerprinted; high-value targets identified (Keycloak admin console, SonarQube,
  ClickHouse prod, n8n, Metabase publicly reachable); TLS posture Grade B (TLS 1.0/1.1
  still offered)
- **Part B (Pentest):** 4 findings against `ledger-api` — YAML deserialization RCE
  (CVSS 9.8, mitigated by Task 1 seccomp), SSRF to internal services confirmed (CVSS 8.6),
  full PAN exposure (PCI DSS violation), version disclosure. Full report in
  [`task4/part-b/pentest-report.pdf`](task4/part-b/pentest-report.pdf)

---

## Repository Structure

```
.github/workflows/
└── secure-pipeline.yml     # Task 2: 4-gate CI/CD pipeline

app/                        # ledger-api source (from starter template)
deploy/
├── namespace.yaml
├── deployment.yaml         # Task 1: hardened (non-root, read-only fs, seccomp, SA)
├── service.yaml
├── configmap.yaml
├── rbac.yaml               # Task 1: least-privilege Role + RoleBinding
├── ingress.yaml
├── neighbour.yaml          # Task 1: hardened reporting service
├── secrets/
│   └── ledger-api-sealedsecret.yaml   # Task 1: Sealed Secret (git-safe)
├── kyverno/
│   └── policies.yaml       # Task 1: admission guardrails (scoped to payments ns)
├── gitops/
│   └── application.yaml    # Task 2: ArgoCD Application manifest
└── istio/
    ├── peer-authentication.yaml    # Task 3: mTLS STRICT
    ├── authorization-policy.yaml   # Task 3: default-deny + SPIFFE-based allow
    └── network-policy.yaml         # Task 3: L3/L4 defense-in-depth

task1/README.md             # Task 1 write-up + evidence
task2/README.md             # Task 2 write-up + pipeline evidence
task3/README.md             # Task 3 write-up + mTLS/authz proof
task4/
├── part-a/README.md        # Recon report
└── part-b/
    ├── README.md           # Pentest summary
    └── pentest-report.pdf  # Full pentest report (PDF)
```

---

## Environment

- Local `kind` cluster (`dodo-assignment`), Kubernetes v1.30.0
- Istio 1.22.5 (latest compatible with K8s 1.30)
- GitHub Actions free-tier runners + GHCR (no cloud account)
- All tooling free and local: kind, kubectl, istioctl, kubeseal, ArgoCD, Kyverno
