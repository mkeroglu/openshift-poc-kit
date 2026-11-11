# 09 — Micro-Segmentation Demo Guide (Tier/Role-Based Isolation)

> 🇹🇷 [Türkçe](README.tr.md) · [← 08 — NetworkPolicy](../08-network-policy/README.md) · [POC flow](../../README.md) · [10 — Multus CNI →](../10-multus-cni/README.md)

This document describes a micro-segmentation scenario that isolates workloads **inside the same namespace** by **tier/role** (the `tier` label) so that **only allowed flows** get through. The demo in [08 — NetworkPolicy](../08-network-policy/README.md) was **namespace-based** (`frontend`/`backend`/`monitoring` were separate namespaces, using `namespaceSelector`); the key difference here is that isolation is applied **within a single namespace** based on the pods' `tier` label (`podSelector`) — the unit of isolation is **the workload's role**, not the namespace boundary. Using the classic 3-tier architecture (**frontend → backend → database**), we prove live that the frontend cannot **bypass** the backend and reach the database directly.

Every step was **tested live on a lab cluster (OpenShift 4.22, OVNKubernetes) in the `poc-microseg-demo` namespace**. The YAMLs are also in this folder as separate files.

Scenario order:

1. Concept — why "micro"?
2. Setting up the demo environment (3 tiers, one namespace)
3. Baseline test (no policy — everything can reach everything)
4. Micro-segmentation NetworkPolicies
5. Verification (actual test output — the critical test: the frontend cannot bypass to the database)
6. Cleanup

---

## 1. Concept — Why "Micro"?

- **Macro-segmentation** (see the [08 — NetworkPolicy](../08-network-policy/README.md) demo): the unit of isolation is the **namespace** — coarse rules like "no pod in this namespace may reach pods in that namespace".
- **Micro-segmentation**: the unit of isolation is **the individual pod/workload role** — even within the same namespace, only **a specific role** (such as `tier: frontend`, `tier: backend`, `tier: database`) may reach **another specific role**, and everything else (even in the same namespace) is rejected.
- Why it matters in practice: even if an attacker or a faulty service gets into the **frontend** tier, it must not be able to **bypass the backend and reach the database directly**. This breaks the assumption "once you're in the right namespace you can reach everything" — the foundation of a **zero-trust** approach.

---

## 2. Setting Up the Demo Environment

Three tiers in a single namespace (each separated by a `tier` label):

```yaml
# tiers.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: poc-microseg-demo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: poc-microseg-demo
  labels:
    tier: frontend
spec:
  replicas: 1
  selector:
    matchLabels: { tier: frontend }
  template:
    metadata:
      labels: { tier: frontend }
    spec:
      containers:
        - name: web
          image: busybox:1.36
          command: ["sh", "-c", "mkdir -p /tmp/www && echo FRONTEND-OK > /tmp/www/index.html && httpd -f -p 8080 -h /tmp/www"]
          ports: [{ containerPort: 8080 }]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: poc-microseg-demo
  labels:
    tier: backend
spec:
  replicas: 1
  selector:
    matchLabels: { tier: backend }
  template:
    metadata:
      labels: { tier: backend }
    spec:
      containers:
        - name: web
          image: busybox:1.36
          command: ["sh", "-c", "mkdir -p /tmp/www && echo BACKEND-OK > /tmp/www/index.html && httpd -f -p 8080 -h /tmp/www"]
          ports: [{ containerPort: 8080 }]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
  namespace: poc-microseg-demo
  labels:
    tier: database
spec:
  replicas: 1
  selector:
    matchLabels: { tier: database }
  template:
    metadata:
      labels: { tier: database }
    spec:
      containers:
        - name: web
          image: busybox:1.36
          command: ["sh", "-c", "mkdir -p /tmp/www && echo DATABASE-OK > /tmp/www/index.html && httpd -f -p 8080 -h /tmp/www"]
          ports: [{ containerPort: 8080 }]
---
apiVersion: v1
kind: Service
metadata: { name: frontend, namespace: poc-microseg-demo }
spec: { selector: { tier: frontend }, ports: [{ port: 8080 }] }
---
apiVersion: v1
kind: Service
metadata: { name: backend, namespace: poc-microseg-demo }
spec: { selector: { tier: backend }, ports: [{ port: 8080 }] }
---
apiVersion: v1
kind: Service
metadata: { name: database, namespace: poc-microseg-demo }
spec: { selector: { tier: database }, ports: [{ port: 8080 }] }
```

```bash
oc apply -f tiers.yaml
oc get deployment -n poc-microseg-demo
```

✅ **Actual output:** `frontend`, `backend`, `database` — all three `1/1 READY`.

---

## 3. Baseline Test (No Policy)

Before applying any NetworkPolicy, confirm that the default behavior is **allow-all**:

```bash
FRONTEND_POD=$(oc get pod -n poc-microseg-demo -l tier=frontend -o jsonpath='{.items[0].metadata.name}')
BACKEND_POD=$(oc get pod -n poc-microseg-demo -l tier=backend -o jsonpath='{.items[0].metadata.name}')

oc exec "$FRONTEND_POD" -n poc-microseg-demo -- wget -qO- --timeout=3 backend.poc-microseg-demo.svc.cluster.local:8080
oc exec "$BACKEND_POD"  -n poc-microseg-demo -- wget -qO- --timeout=3 database.poc-microseg-demo.svc.cluster.local:8080
oc exec "$FRONTEND_POD" -n poc-microseg-demo -- wget -qO- --timeout=3 database.poc-microseg-demo.svc.cluster.local:8080
```

✅ **Actual output:** all three succeed (`BACKEND-OK`, `DATABASE-OK`, `DATABASE-OK`) — **with no policy, the frontend can also reach the database directly**. This is exactly the problem the NetworkPolicies below solve.

---

## 4. Micro-Segmentation NetworkPolicies

Three rules: **(1)** deny everything by default, **(2)** allow only frontend→backend, **(3)** allow only backend→database. All use **`podSelector`** (not `namespaceSelector`) — the unit of isolation is the role, not the namespace.

```yaml
# microseg-policies.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: poc-microseg-demo
spec:
  podSelector: {}          # ALL pods in the namespace
  policyTypes: [Ingress]
  # no ingress rule -> no traffic allowed (default deny)
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: poc-microseg-demo
spec:
  podSelector:
    matchLabels: { tier: backend }     # this rule protects ONLY backend pods
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector:
            matchLabels: { tier: frontend }   # ONLY from frontend
      ports:
        - protocol: TCP
          port: 8080
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-database
  namespace: poc-microseg-demo
spec:
  podSelector:
    matchLabels: { tier: database }    # this rule protects ONLY database pods
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector:
            matchLabels: { tier: backend }    # ONLY from backend
      ports:
        - protocol: TCP
          port: 8080
```

```bash
oc apply -f microseg-policies.yaml
```

> ⚠️ **Watch out — Route traffic is cut too:** without an "allow" rule for `frontend`, `default-deny-ingress` also cuts **external traffic coming from the OpenShift router (Route)**; the application goes dark for users.
>
> ✅ **Actual output:** before the policies, `curl http://<frontend-route>` → `200`; after applying `microseg-policies.yaml` → `000` (timeout).

Fix: a rule that allows traffic to the frontend only from the namespace where the router runs (`allow-ingress-to-frontend.yaml`). OpenShift marks the ingress controller namespace with the label `policy-group.network.openshift.io/ingress: ""`:

```yaml
# allow-ingress-to-frontend.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-to-frontend
  namespace: poc-microseg-demo
spec:
  podSelector:
    matchLabels: { tier: frontend }
  policyTypes: [Ingress]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              policy-group.network.openshift.io/ingress: ""
      ports:
        - protocol: TCP
          port: 8080
```

```bash
oc expose svc frontend -n poc-microseg-demo
oc apply -f allow-ingress-to-frontend.yaml
```

✅ **Actual output:** Route → `FRONTEND-OK [200]`. Even when a Route was opened for `database`, external access stayed `000`, because the router was allowed only for the frontend.

---

## 5. Verification (actual test output)

**Repeat the same 3 tests with the policies in place:**

```bash
echo "frontend -> backend:"
oc exec "$FRONTEND_POD" -n poc-microseg-demo -- wget -qO- --timeout=3 backend.poc-microseg-demo.svc.cluster.local:8080

echo "backend -> database:"
oc exec "$BACKEND_POD" -n poc-microseg-demo -- wget -qO- --timeout=3 database.poc-microseg-demo.svc.cluster.local:8080

echo "frontend -> database (DIRECT, bypassing backend):"
oc exec "$FRONTEND_POD" -n poc-microseg-demo -- wget -qO- --timeout=3 database.poc-microseg-demo.svc.cluster.local:8080
```

✅ **Actual output:**

```
frontend -> backend:
BACKEND-OK                          <-- allowed flow, works

backend -> database:
DATABASE-OK                         <-- allowed flow, works

frontend -> database (DIRECT, bypassing backend):
wget: download timed out            <-- CRITICAL TEST: blocked
command terminated with exit code 1
```

**Result: even though the frontend and database are in the same namespace on the same cluster network, communication between them is completely cut because there is no `podSelector`-based rule between them.** The frontend cannot reach the database **in any way** by bypassing the backend — this is the core promise of micro-segmentation.

> **Note:** don't test with a pod that has no `tier` label (e.g. a generic "tester" pod) — `default-deny-ingress` (rightly) blocks it too and gives a misleading "everything is broken" result. Always test **with `oc exec` from inside the actual role pods**, and remember that the identity used in the test is subject to the segmentation as well.

---

## 6. Cleanup

```bash
oc delete project poc-microseg-demo
```

---

## Summary

| Allowed flow | Which NetworkPolicy | Test result |
|---|---|---|
| frontend → backend :8080 | `allow-frontend-to-backend` | ✅ `BACKEND-OK` |
| backend → database :8080 | `allow-backend-to-database` | ✅ `DATABASE-OK` |
| frontend → database :8080 (direct) | *(none — deliberately absent)* | ❌ timeout |
| Outside world (Route) → frontend | `allow-ingress-to-frontend` | ✅ `FRONTEND-OK` (`000` without the rule) |
| Outside world (Route) → database | *(none)* | ❌ `000` |

**Golden rules:**
1. In micro-segmentation the unit of isolation is **the workload's role, not the namespace** (`podSelector`, not `namespaceSelector`).
2. "We're in the same namespace" is no security guarantee — without `default-deny-ingress` + role-specific `allow` rules, every pod in a namespace can reach every other one freely.
3. `default-deny-ingress` cuts Route traffic too: for the externally exposed tier (frontend), traffic from the router namespace (`policy-group.network.openshift.io/ingress`) must be allowed explicitly. Tiers that must not be exposed (database) get no such rule.
4. Always test **from inside the pod of the relevant role** — a test pod that is not subject to the segmentation (no label) gives misleading "everything is broken" results.
