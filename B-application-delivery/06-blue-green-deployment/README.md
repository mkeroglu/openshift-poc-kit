# 06 — Blue/Green Deployment Demo Guide

> 🇹🇷 [Türkçe](README.tr.md) · [← 05 — CI/CD](../05-ci-cd/README.md) · [POC flow](../../README.md) · [07 — Canary Deployment →](../07-canary-deployment/README.md)

This document covers the **Blue/Green deployment** strategy on OpenShift. Think of it as a continuation of, or an alternative to, the pipeline automation in [05 — CI/CD](../05-ci-cd/README.md) — there we updated a single Deployment in place (rolling) with `oc rollout restart`; here we keep **two independent environments** running at the same time and switch between them instantly. Every step was **tested live on a lab cluster (OpenShift 4.22) in the `poc-bg-demo` namespace**. All manifests are in `blue-green-demo.yaml`.

Scenario order:

1. Concept
2. Implementation: two Deployments + one Service + Route
3. Verification (actual test output — switch and rollback)
4. A real pitfall: router delay
5. Cleanup

---

## 1. Concept

In a Blue/Green deployment, **two separate, fully independent** environments (Deployments) are kept running at the same time — one taking traffic right now (**e.g. Blue**), the other running the new version but not yet taking traffic (**e.g. Green**). The switch happens **instantly, in one step**, by pointing a single **Service selector** (or Route target) from Blue to Green — unlike a rolling update, old and new pods never share mixed traffic; the switch works like a clean "toggle". The same mechanism in reverse gives an **instant rollback** (if Green has a problem, point the selector back to Blue; no redeploy needed).

Unlike the `oc rollout restart` model (one Deployment, rolling update) used by `pipeline-build-deploy.yaml` in [05 — CI/CD](../05-ci-cd/README.md), Blue/Green needs **two separate Deployments** and **one Service** that ties them together.

---

## 2. Implementation: Two Deployments + One Service + Route

```yaml
# blue-green-demo.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: poc-bg-demo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-blue
  namespace: poc-bg-demo
  labels:
    app: demo-app
    version: blue
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo-app
      version: blue
  template:
    metadata:
      labels:
        app: demo-app
        version: blue
    spec:
      containers:
        - name: web
          image: busybox:1.36
          command: ["sh", "-c", "mkdir -p /tmp/www && echo BLUE > /tmp/www/index.html && httpd -f -p 8080 -h /tmp/www"]
          ports:
            - containerPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-green
  namespace: poc-bg-demo
  labels:
    app: demo-app
    version: green
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo-app
      version: green
  template:
    metadata:
      labels:
        app: demo-app
        version: green
    spec:
      containers:
        - name: web
          image: busybox:1.36
          command: ["sh", "-c", "mkdir -p /tmp/www && echo GREEN > /tmp/www/index.html && httpd -f -p 8080 -h /tmp/www"]
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: demo-app
  namespace: poc-bg-demo
spec:
  selector:
    app: demo-app
    version: blue        # <-- the ONLY field that routes traffic; the Blue/Green switch is changing this
  ports:
    - port: 8080
      targetPort: 8080
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: demo-app
  namespace: poc-bg-demo
spec:
  to:
    kind: Service
    name: demo-app
  port:
    targetPort: 8080
```

```bash
oc apply -f blue-green-demo.yaml
oc get deployment -n poc-bg-demo
```

✅ **Actual output:** both `app-blue` and `app-green` are `2/2 READY` — the two environments are up **at the same time**, and only the one selected by the Service gets traffic.

---

## 3. Verification (actual test output)

**a) Initial state — the Service selects Blue:**

```bash
HOST=$(oc get route demo-app -n poc-bg-demo -o jsonpath='{.spec.host}')
curl -s "http://$HOST"
```

✅ `BLUE`

**b) Switch — point the Service selector to Green:**

```bash
oc patch service demo-app -n poc-bg-demo --type merge \
  -p '{"spec":{"selector":{"app":"demo-app","version":"green"}}}'
```

```bash
for i in 1 2 3 4 5; do curl -s "http://$HOST"; echo; done
```

✅ **Actual output:** time until the first `GREEN` response after the switch, measured with requests sent 0.1–0.2 s apart (4 rounds):

| Round | Blue → Green | Green → Blue (rollback) | Failed/empty responses |
|---|---|---|---|
| 1 | 1.67 s | 2.36 s | 0 |
| 2 | 0.25 s | 0.14 s | 0 |
| 3 | 0.25 s | 0.14 s | 0 |
| 4 | 0.25 s | 4.02 s | 0 |

**No request failed** during the switch; the old version answered until the switch moment, the new version after it. The duration, however, is not constant (see section 4).

**c) Rollback — go back to Blue instantly:**

```bash
oc patch service demo-app -n poc-bg-demo --type merge \
  -p '{"spec":{"selector":{"app":"demo-app","version":"blue"}}}'
```

✅ **Actual output:** the rollback also completed within 0.14–4 s with a single `oc patch`, with no redeploy/build (table above).

---

## 4. A Real Pitfall: Router Delay

Not at the moment of the switch — the OpenShift router (HAProxy) picking up the new `Endpoints` and updating its backend pool is **not instant** and **varies**: measured between 0.14 s and 4 s in the live tests (~5 s was also seen in an earlier round). The root cause is not a bug; it is the natural (small) delay of the router noticing the endpoint change and reloading the HAProxy config.

**Practical takeaway:** before considering a Blue/Green switch "done", verify that all traffic has really moved to the new version by testing again a few seconds apart — especially in automation (CI/CD scripts), leave a fixed wait (`sleep`) after the switch and then verify; don't move on assuming "the patch command returned = traffic has moved".

---

## 5. Cleanup

```bash
oc delete namespace poc-bg-demo
```

---

## Summary

| Component | Role |
|---|---|
| `app-blue` / `app-green` Deployment | Two independent environments running at the same time |
| `Service.spec.selector` | The single switch that routes traffic — `version: blue` ↔ `version: green` |
| `Route` | External access, targets the Service, does not change during the switch |

**Golden rules:**
1. Unlike a rolling update, Blue/Green needs **twice the resources** (CPU/memory) — both environments run at the same time.
2. Switch = one `Service.spec.selector` change; rollback is the same mechanism in reverse.
3. The router's endpoint update after the switch is **not instant** — allow for a few seconds of delay and verify; don't assume "done" right away.
