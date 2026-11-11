# 07 — Canary Deployment Demo Guide

> 🇹🇷 [Türkçe](README.tr.md) · [← 06 — Blue/Green Deployment](../06-blue-green-deployment/README.md) · [POC flow](../../README.md) · [08 — NetworkPolicy →](../../C-network-and-security/08-network-policy/README.md)

This document covers the **Canary deployment** strategy on OpenShift. The difference from [06 — Blue/Green Deployment](../06-blue-green-deployment/README.md): traffic does not move 100% at once, it is shifted to the new version **in gradual percentages** (`10% → 50% → 100%`) — so that if the new version has a problem only a small share of users is affected, and going back (lowering the percentage) is easy. No extra component (Service Mesh, etc.) is needed: it is done directly with the **OpenShift Route**'s `alternateBackends` + `weight` fields, using the HAProxy router's own traffic-splitting mechanism. Every step was **tested live on a lab cluster (OpenShift 4.22) in the `poc-canary-demo` namespace** — at each stage 100–200 real requests were sent and the distribution was actually measured. All manifests are in `canary-demo.yaml`.

Scenario order:

1. Concept
2. Implementation: two Deployments + two Services + a weighted Route
3. Verification (actually measured distributions — 90/10, 50/50, 100/0, rollback)
4. Notes
5. Cleanup

---

## 1. Concept

- In **Blue/Green**, traffic moves **100% in one step** by changing `Service.spec.selector`.
- In **Canary**, traffic is **split by weight** through a **Route** between **two different Services that are up at the same time** (stable + canary). On an OpenShift Route this is defined with `spec.to.weight` (primary backend) and `spec.alternateBackends[].weight` (secondary backends) — the router distributes each incoming request between the two Services probabilistically (weighted round-robin/random) according to these weights.
- Typical flow: start with a small percentage (**10%**) → watch metrics/error rate → if all is well, raise the percentage gradually (**50%**) → finally go to **100%** (full promotion). If a problem shows up at any stage, just drop the percentage to **0** — no redeploy is needed for rollback.

---

## 2. Implementation: Two Deployments + Two Services + Weighted Route

```yaml
# canary-demo.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: poc-canary-demo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-stable
  namespace: poc-canary-demo
  labels:
    app: demo-app
    version: stable
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo-app
      version: stable
  template:
    metadata:
      labels:
        app: demo-app
        version: stable
    spec:
      containers:
        - name: web
          image: busybox:1.36
          command: ["sh", "-c", "mkdir -p /tmp/www && echo STABLE > /tmp/www/index.html && httpd -f -p 8080 -h /tmp/www"]
          ports:
            - containerPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-canary
  namespace: poc-canary-demo
  labels:
    app: demo-app
    version: canary
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo-app
      version: canary
  template:
    metadata:
      labels:
        app: demo-app
        version: canary
    spec:
      containers:
        - name: web
          image: busybox:1.36
          command: ["sh", "-c", "mkdir -p /tmp/www && echo CANARY > /tmp/www/index.html && httpd -f -p 8080 -h /tmp/www"]
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: app-stable
  namespace: poc-canary-demo
spec:
  selector:
    app: demo-app
    version: stable
  ports:
    - port: 8080
      targetPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: app-canary
  namespace: poc-canary-demo
spec:
  selector:
    app: demo-app
    version: canary
  ports:
    - port: 8080
      targetPort: 8080
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: demo-app
  namespace: poc-canary-demo
spec:
  to:
    kind: Service
    name: app-stable
    weight: 90              # <-- 90% of traffic to stable
  alternateBackends:
    - kind: Service
      name: app-canary
      weight: 10             # <-- 10% to canary
  port:
    targetPort: 8080
```

```bash
oc apply -f canary-demo.yaml
oc get deployment -n poc-canary-demo
```

✅ **Actual output:** both `app-stable` and `app-canary` are `2/2 READY` — like Blue/Green, two environments are up at the same time; the difference is that traffic goes to **two separate Services** (weighted through the Route) instead of **one Service**.

---

## 3. Verification (actually measured distributions)

The same test at each stage: **100 real `curl` requests** were sent and the `STABLE`/`CANARY` responses counted.

```bash
HOST=$(oc get route demo-app -n poc-canary-demo -o jsonpath='{.spec.host}')
STABLE=0; CANARY=0
for i in $(seq 1 100); do
  R=$(curl -s "http://$HOST")
  if [ "$R" = "STABLE" ]; then STABLE=$((STABLE+1)); elif [ "$R" = "CANARY" ]; then CANARY=$((CANARY+1)); fi
done
echo "STABLE: $STABLE  CANARY: $CANARY"
```

### a) Start: `90% stable / 10% canary`

✅ **Actual output (200 requests):** `STABLE: 181  CANARY: 19` (90.5% / 9.5%) — consistent with the configured ratio. An earlier test round measured `91 / 9` over 100 requests.

### b) Raise: `50% / 50%`

```bash
oc patch route demo-app -n poc-canary-demo --type=json -p '[
  {"op":"replace","path":"/spec/to/weight","value":50},
  {"op":"replace","path":"/spec/alternateBackends/0/weight","value":50}
]'
```

✅ **Actual output:** `STABLE: 50  CANARY: 50` — exactly half and half.

### c) Full promotion: `0% stable / 100% canary`

```bash
oc patch route demo-app -n poc-canary-demo --type=json -p '[
  {"op":"replace","path":"/spec/to/weight","value":0},
  {"op":"replace","path":"/spec/alternateBackends/0/weight","value":100}
]'
```

✅ **Actual output (100 requests):** `STABLE: 0  CANARY: 100` — with `weight: 0`, **no** requests go to stable; full cut-over verified.

### d) Rollback: instantly `100% stable / 0% canary`

```bash
oc patch route demo-app -n poc-canary-demo --type=json -p '[
  {"op":"replace","path":"/spec/to/weight","value":100},
  {"op":"replace","path":"/spec/alternateBackends/0/weight","value":0}
]'
```

✅ **Actual output (100 requests):** `STABLE: 100  CANARY: 0` — back to the old version instantly with a single `oc patch`, no redeploy. Failed/empty responses at every stage: **0**.

---

## 4. Notes

- **Compared with the router delay in Blue/Green:** in the Blue/Green demo (`Service.spec.selector` change) the switch took 0.1–4 s to reach the router. For Canary (`Route.spec` weight change) we waited 3 s after each change before measuring; the new ratio was consistent from the first request. Still, in automation it is a safe habit to leave a short wait after each weight change before measuring/deciding.
- **Weights are relative**, not absolute percentages: `weight: 90` / `weight: 10` and `weight: 9` / `weight: 1` give the same 90%/10% ratio mathematically — the router normalizes by the sum of the weights.
- **Sticky sessions:** on HTTP/edge routes the OpenShift router sets a sticky cookie **by default** (`set-cookie: <hash>=...; HttpOnly`). Clients that keep the cookie (browsers) are assigned to a version by weight on the first request and **then always stay on that version**; for canary this is the desired behavior, so users don't bounce between versions. Clients that don't keep cookies (curl, some API clients) are redistributed on every request.

  ✅ **Actual output (50%/50%):**

  | Client | Result of 20 requests |
  |---|---|
  | Cookie-keeping clients 1, 2, 5 | 20/20 `CANARY` |
  | Cookie-keeping clients 3, 4 | 20/20 `STABLE` |
  | Client without cookies | 10 `STABLE` / 10 `CANARY` |
  | `haproxy.router.openshift.io/balance=source` + `disable_cookies=true` (single client IP, 40 requests) | 40/40 `STABLE` |

  `balance=source` sticks by client IP. If users are behind NAT/a proxy, many users come from the same IP and the distribution is skewed; in that case prefer cookies.
- For more advanced canary scenarios (automatic metric-based promotion/rollback, routing by HTTP header/user), **OpenShift Service Mesh** (Istio) is a better fit; the method here is the native OpenShift Route mechanism, which needs no extra component.

---

## 5. Cleanup

```bash
oc delete namespace poc-canary-demo
```

---

## Summary

| Component | Role |
|---|---|
| `app-stable` / `app-canary` Deployment + Service | Two independent environments, **two separate** Services (Blue/Green used one Service) |
| `Route.spec.to.weight` | Relative traffic share of the primary backend (stable) |
| `Route.spec.alternateBackends[].weight` | Relative traffic share of the secondary backend (canary) |

**Golden rules:**
1. Canary is the gradual form of Blue/Green — the same "keep two environments up at once" principle, but traffic is split with **Route weights** instead of `Service.selector`.
2. Weights are **relative**, not absolute — `weight: 0` disables a backend completely.
3. After each weight change, verify **by measuring with real traffic** — "the patch command succeeded" ≠ "traffic is split at the intended ratio".
4. By default the router is sticky via cookie: browser users stick to one version, clients without cookies are redistributed on every request. Account for this in tests.
