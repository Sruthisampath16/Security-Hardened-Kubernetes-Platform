# Security-Hardened Kubernetes Platform

Cluster-level security hardening applied to the workloads deployed in [flask-k8s-cicd](../flask-k8s-cicd) — RBAC, NetworkPolicy, Pod Security Standards, and image scanning layered on top of an existing Flask + Postgres application.

## What this project demonstrates

- Role-Based Access Control (RBAC): least-privilege identities and permissions, verified with `kubectl auth can-i`
- Network segmentation between workloads using NetworkPolicy
- Pod Security Standards enforcement at the namespace level, including a real fix for a non-obvious Kubernetes verification gap
- Container image vulnerability scanning with a measured before/after improvement
- An honest understanding of the difference between *defining* a security control and a cluster actually *enforcing* it

## Project structure

```
k8s-security-hardening/
├── namespace.yaml
├── service-acc.yaml
├── role.yaml
├── role-bind.yaml
├── network-policies.yaml
└── README.md
```

## 1. RBAC — least-privilege access control

- Dedicated namespace (`rbac-file`) isolating this demo from the real application
- ServiceAccount (`readonly-sa`) with a Role (`pod-reader`) granting only `get`, `list`, `watch` on pods — no `create`/`update`/`delete`
- A RoleBinding connecting the two

**Verification:**
```bash
kubectl auth can-i get pods --as=system:serviceaccount:rbac-file:readonly-sa -n rbac-file
# yes
kubectl auth can-i delete pods --as=system:serviceaccount:rbac-file:readonly-sa -n rbac-file
# no
```

**Real-world application:** the same pattern would scope a CI/CD pipeline's kubeconfig to only update Deployments in one namespace, rather than holding cluster-admin — limiting the blast radius if that credential ever leaked.

## 2. NetworkPolicy — restricting pod-to-pod traffic

A policy restricting Postgres to only accept connections from Flask pods, on port 5432.

**Before:** any pod (tested with an unrelated nginx pod) could reach Postgres directly — connection succeeded (curl exit code 52, "empty reply," meaning TCP connected fine, Postgres just doesn't speak HTTP).

**Known limitation, documented honestly:** after applying the policy, traffic was still not blocked. Root cause: Docker Desktop's Kubernetes uses **kindnet** as its CNI, which does not implement NetworkPolicy enforcement (confirmed by checking `kube-system` — no Calico/Cilium/Weave present). The policy is correctly defined and visible via `kubectl get networkpolicy`, but nothing in this cluster's networking layer enforces it. This exact policy would be enforced without changes on EKS or with Calico installed locally.

## 3. Pod Security Standards — enforced, with a real fix required

```bash
kubectl label namespace default pod-security.kubernetes.io/enforce=restricted
```

Unlike NetworkPolicy, this **is** enforced directly by the API server. Applying it immediately flagged existing pods, and a subsequent rollout restart **genuinely failed**:

```
Error: container has runAsNonRoot and image has non-numeric user (appuser),
cannot verify user is non-root
```

**The gap:** the Dockerfile already uses `USER appuser` (non-root), but Kubernetes' `runAsNonRoot: true` check cannot verify a *named* user is safe — it requires a numeric UID it can confirm is non-zero.

**The fix** — found the actual UID (`docker run ... id appuser` → `100`) and made it explicit in `deployment.yaml`:
```yaml
securityContext:            # pod-level
  runAsNonRoot: true
  runAsUser: 100
  seccompProfile:
    type: RuntimeDefault
```
```yaml
securityContext:            # container-level
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
```

After this fix, all 3 Flask pods rolled out successfully under the `restricted` standard, with `/readyz` and database connectivity confirmed still working.

## 4. Image scanning — Docker Scout

```bash
docker scout quickview sruthidoc07/flask-k8s-cicd:latest
```

**Baseline result:**
| Severity | Count |
|---|---|
| Critical | 1 |
| High | 4 |
| Medium | 7 |
| Low | 31 |
| Health score | D (44%) |

Scout identified the vulnerabilities as almost entirely inherited from a stale pull of the `python:3.12-slim` base image, not from application code.

**Action taken:** rebuilt the image with `docker build --no-cache`, forcing a fresh pull of the base image layer (base image tags are patched over time under the same name — an old local build can lag behind the current, more-patched version).

**Result after rebuild and push:**
| Severity | Count |
|---|---|
| Critical | 0 |
| High | 1 |
| Medium | 6 |
| Low | 26 |
| Health score | C (67%) |

The "Fixable critical or high vulnerabilities" policy flipped from failing to passing — everything that had an available fix was resolved with zero code changes, purely by refreshing the base layer.

