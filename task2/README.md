# Task 2 — Secure CI/CD Pipeline & Supply Chain

## Overview
A GitHub Actions pipeline (`.github/workflows/secure-pipeline.yml`) that builds, scans, signs,
and (on success) pushes `ledger-api` to GHCR. Security is enforced by the **pipeline**, not
by good intentions: 4 independent gates run before an image is ever built or pushed, each
with an explicit, documented fail policy.

**Environment:** GitHub-hosted runners (free tier) + GHCR — no cloud account required.

## Pipeline stages

```
push to main
     │
     ├─▶ Gate 1: Secrets Scan (gitleaks)         ──▶ HARD BLOCK on any finding
     ├─▶ Gate 2: SAST (Semgrep)                  ──▶ HARD BLOCK on HIGH/CRITICAL
     └─▶ Gate 3: Dependency Scan (Trivy, fs)      ──▶ HARD BLOCK on fixable CRITICAL
              │  (all three gates must pass)
              ▼
          Build Image (tagged with git SHA, never :latest)
              │
              ▼
     Gate 4: Container Image Scan (Trivy, image)  ──▶ HARD BLOCK on fixable CRITICAL
              │
              ▼
     Push to GHCR ──▶ Cosign sign (keyless, OIDC) ──▶ SLSA provenance attestation
```

## Fail policy per gate

| Gate | Tool | Hard blocks on | Warns only on |
|---|---|---|---|
| 1. Secrets | gitleaks | Any finding, no exceptions | — |
| 2. SAST | Semgrep (`p/security-audit`, `p/python`, `p/owasp-top-ten`) | ERROR-severity (HIGH/CRITICAL) findings | WARNING/INFO findings (visible via SARIF in Security tab only) |
| 3. Dependency scan | Trivy (filesystem, `./app`) | CRITICAL CVE **with a fix available** | HIGH/MEDIUM (SARIF only); CRITICAL with **no fix yet** (`--ignore-unfixed`) |
| 4. Image scan | Trivy (built image) | Same policy as Gate 3, applied to the final image | Same as Gate 3 |

**Why "ignore-unfixed" for CRITICAL CVEs with no patch:** blocking indefinitely on a CVE that
has no available fix just stalls delivery without reducing risk — the team can't act on it
by patching. These are logged and tracked (visible in the SARIF upload to the Security tab)
rather than gating the pipeline, so they don't create a permanent deadlock.

## Real findings from this pipeline (run #1, evidence)

The pipeline was run against the actual `ledger-api` starter code and **correctly blocked
the build** at two independent gates — this is the pipeline working as intended, not a bug.

### Gate 2 (Semgrep) — 4 blocking findings, all in `app/app.py`

```
python.flask.security.insecure-deserialization.insecure-deserialization
  Line 39: config = yaml.load(request.data)
  → Detected use of an insecure deserialization library. Prone to code execution.

python.django.security.injection.ssrf.injection-requests.ssrf-injection-requests
python.flask.security.injection.ssrf-requests.ssrf-requests
  Line 45-46: url = request.args.get("url", ""); resp = requests.get(url, timeout=5)
  → Data from the request object passed to a new server-side request (SSRF).
```

Ran 71 rules on 3 files → 4 findings, all blocking. Exit code 1 → pipeline halted.

### Gate 3 (Trivy) — 3 CRITICAL CVEs in `requirements.txt`

```
requirements.txt (pip) — Total: 3 (CRITICAL: 3)

CVE-2019-20477  PyYAML 5.1  CRITICAL  fixed in 5.2    — command execution via python/object/apply
CVE-2020-14343  PyYAML 5.1  CRITICAL  fixed in 5.4    — incomplete fix for CVE-2020-1747
CVE-2020-1747   PyYAML 5.1  CRITICAL  fixed in 5.3.1  — arbitrary command execution via python/object/new
```

**Notable:** SAST and SCA independently caught the *same root cause* from two different
angles — Semgrep flagged the dangerous `yaml.load()` call pattern in the code, while Trivy
flagged the vulnerable PyYAML version that makes that call exploitable. This is
defense-in-depth working as intended: either gate alone would have caught this class of bug.

**Remediation (not yet applied, tracked as follow-up):** upgrade `PyYAML` to `>=6.0` and
replace `yaml.load(request.data)` with `yaml.safe_load(request.data)`, and either remove or
strictly allow-list the `/fetch?url=` endpoint to prevent SSRF.

## Cosign signing & provenance

- Images are signed **keylessly** using GitHub's OIDC identity (`id-token: write` permission) —
  no long-lived signing keys stored as secrets.
- A SLSA-style provenance predicate (build source, commit SHA, workflow identity, run ID) is
  attested alongside the signature via `cosign attest`.
- `cosign verify` runs as a sanity check in the same job, confirming the signature is valid
  and was issued by *this* repository's workflow before considering the push "complete."
- This signature is what Task 1's `require-signed-images` Kyverno policy checks against —
  once this pipeline successfully produces a signed image, only that image (not any
  arbitrary `ghcr.io/*/ledger-api*` image) can be deployed to the hardened cluster.

## Image tagging

Images are tagged with the **git commit SHA** (`sha-<12-char-sha>`), never `:latest` —
this satisfies Task 1's `disallow-latest-tag` Kyverno policy and ensures every deployed
image is traceable to an exact commit.

## GitOps (ArgoCD)

ArgoCD is installed in-cluster (`argocd` namespace) with an `Application` resource
(`deploy/gitops/application.yaml`) pointing at this repo's `deploy/` folder, targeting the
`payments` namespace. `syncPolicy.automated` has both `prune: true` and `selfHeal: true` —
Kyverno's `disallow-root-containers`/`disallow-latest-tag` policies were scoped to the
`payments` namespace only (see note below) so they don't block ArgoCD's own platform pods.

**Initial sync — proof GitOps works:**
```
NAME         SYNC STATUS   HEALTH STATUS
ledger-api   Synced        Healthy
```
ArgoCD dashboard confirms: repo URL, target revision `main`, path `deploy`, destination
namespace `payments` — all matching this repo exactly.

**Drift detection + self-heal — proof (bonus requirement):**

A manual `kubectl scale deployment ledger-api -n payments --replicas=1` was run to simulate
an out-of-band change (e.g. an engineer bypassing GitOps). ArgoCD detected the drift and
self-healed within seconds:

```
$ kubectl scale deployment ledger-api -n payments --replicas=1
deployment.apps/ledger-api scaled

# ~15s later, without any manual intervention:
$ kubectl get pods -n payments -l app=ledger-api
NAME                          READY   STATUS    RESTARTS   AGE
ledger-api-76db444575-db6x6   1/1     Running   0          44s   ← new pod, self-healed
ledger-api-76db444575-dg4zc   1/1     Running   0          5h13m ← original, untouched
ledger-api-76db444575-qrnp8   1/1     Running   0          44s   ← new pod, self-healed
```

ArgoCD's own event log confirms the automated reconciliation:
```
Updated sync status: Synced -> OutOfSync
Updated health status: Healthy -> Progressing
OperationCompleted  Partial sync operation to 6fbe2024... succeeded
Updated sync status: OutOfSync -> Synced
Updated health status: Progressing -> Healthy
```
Full drift-to-healed cycle completed in ~7 seconds, with zero manual `kubectl apply` needed
to restore the desired state — git remains the single source of truth.

**Note on Kyverno namespace scoping:** the first ArgoCD install attempt was blocked by Task 1's
cluster-wide `disallow-root-containers` policy, since ArgoCD's own controller pods don't set
`runAsNonRoot`. This correctly highlighted that admission policies meant for a PCI-scoped
*application* namespace shouldn't apply to *platform* tooling — the policies were updated to
scope `match.resources.namespaces: [payments]` instead of cluster-wide, and reapplied. See
`deploy/kyverno/policies.yaml` for the corrected version.

## What I'd do with more time
- Fix the PyYAML/yaml.safe_load and SSRF findings above, then re-run the pipeline to show
  a full green run through to Cosign signing.
- Add Grype as a second CVE scanner for cross-validation against Trivy.
- Add a `permissions:` audit step (least-privilege GITHUB_TOKEN scoping per job).
