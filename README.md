# Security-Hardened Kubernetes Platform

Cluster-level security hardening applied to the workloads deployed in [flask-k8s-cicd](../flask-k8s-cicd) — RBAC, NetworkPolicy, and Pod Security controls layered on top of an existing Flask + Postgres application, without modifying the application itself.

## What this project demonstrates

- Role-Based Access Control (RBAC): least-privilege identities and permissions, verified with `kubectl auth can-i`
- Network segmentation between workloads using NetworkPolicy
- An honest understanding of the difference between *defining* a security policy and a cluster actually *enforcing* it
- Practical debugging of real Kubernetes YAML/RBAC issues (documented below, not hidden)

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

**What was built:**
- A dedicated namespace (`rbac-file`) to isolate practice/demo resources from the real application
- A ServiceAccount (`readonly-sa`) — an identity with no permissions by default
- A Role (`pod-reader`) granting only `get`, `list`, `watch` on pods — explicitly excluding `create`, `update`, `delete`
- A RoleBinding connecting the ServiceAccount to the Role

**Verification (not just "it applied without error" — actual proof of enforcement):**
```bash
kubectl auth can-i get pods --as=system:serviceaccount:rbac-file:readonly-sa -n rbac-file
# yes

kubectl auth can-i delete pods --as=system:serviceaccount:rbac-file:readonly-sa -n rbac-file
# no
```

This contrast — allowed for reads, denied for writes — is the actual evidence that RBAC is being enforced correctly, not just configured.

**Real-world application:** this same pattern would apply to CI/CD credentials (e.g., a GitHub Actions kubeconfig scoped to only update a specific Deployment in a specific namespace, rather than holding cluster-admin), monitoring tools that only need read access, or junior engineers who need visibility without write access to production.

## 2. NetworkPolicy — restricting pod-to-pod traffic

**What was built:** a policy restricting the Postgres pod to only accept incoming connections from Flask pods, on port 5432:

```yaml
podSelector:
  matchLabels:
    app: postgres
ingress:
  - from:
      - podSelector:
          matchLabels:
            app: flask-app
    ports:
      - protocol: TCP
        port: 5432
```

**Before applying this policy**, any pod in the namespace — including an unrelated nginx pod used purely as a test — could reach Postgres directly:
```bash
kubectl exec -it <nginx-pod> -- curl -s postgres-service:5432
# empty reply from server (curl exit code 52) — connection succeeded
```

**Known limitation, documented honestly:** after applying the policy, the connection was still not blocked. Investigation confirmed the cause: Docker Desktop's Kubernetes uses **kindnet** as its CNI (Container Network Interface) plugin, which does not implement NetworkPolicy enforcement. The policy object is created correctly and is visible via `kubectl get networkpolicy`, but nothing in this cluster's networking layer actually reads and enforces it.

```bash
kubectl get pods -n kube-system
# kindnet-xxxxx present; no calico/cilium/weave
```

This is a platform limitation, not a configuration error — the identical policy YAML would be enforced without any changes on a cloud cluster like EKS (which supports CNI plugins with NetworkPolicy support), or locally with Calico installed alongside kindnet.


