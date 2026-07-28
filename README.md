# Dodo Payments — DevSecOps Assessment

Submission by Sowmiya Manikandan for the Security & DevOps Engineer assessment.

## Overview
This repo hardens `ledger-api`, a payments microservice, end-to-end: workload security,
secure CI/CD supply chain, zero-trust service mesh networking, and offensive
security (recon + pentest). Everything runs locally and free — a `kind` Kubernetes
cluster plus GitHub Actions, no cloud account required.

## Tasks

| Task | Status | Link |
|---|---|---|
| Task 1 — Deploy & Harden the Workload | ✅ Complete | [task1/README.md](task1/README.md) |
| Task 2 — Secure CI/CD Pipeline & Supply Chain | ⬜ In progress | task2/README.md |
| Task 3 — Service Mesh & Zero-Trust (Istio) | ⬜ Pending | task3/README.md |
| Task 4 — Recon & Penetration Testing | ⬜ Pending | task4/README.md |

## Repo structure
```
deploy/          # Kubernetes manifests for Task 1 (hardened ledger-api)
task1/           # Task 1 write-up, evidence, screenshots
app/             # ledger-api source (from starter template)
.github/         # CI/CD workflows (Task 2)
```

## Environment
- Local `kind` Kubernetes cluster (`dodo-assignment`), Kubernetes v1.30.0
- No cloud account used — all tooling free/local
