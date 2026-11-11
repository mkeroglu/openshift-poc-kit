# 08 — Kubernetes / OpenShift NetworkPolicy Demo Guide

> 🇹🇷 [Türkçe](README.tr.md) · [← 07 — Canary Deployment](../../B-application-delivery/07-canary-deployment/README.md) · [POC flow](../../README.md) · [09 — Micro-Segmentation →](../09-micro-segmentation/README.md)

This document is written for presenting NetworkPolicy live (inside pods, with `oc`). Every command and YAML was **tested end to end on a lab cluster** (OpenShift 4.22, OVNKubernetes), and the output was verified as shown below. The YAMLs are also in this folder as separate files. Four namespaces are used: `poc-np-frontend`, `poc-np-backend`, `poc-np-other`, `poc-np-monitoring`.

Scenario order:

1. What NetworkPolicy is and why it is needed
2. Prerequisite check (CNI/SDN support)
3. Setting up the demo environment (4 namespaces + backend service + test pods)
4. Scenario 1 — Block all ingress traffic in a namespace (deny-all)
5. Scenario 2 — Allow traffic only from a specific namespace (+ add a second namespace to show the additive behavior)
6. Scenario 3 — Allow traffic only from a specific pod / label
7. Scenario 4 — Restrict egress traffic + a real pitfall
8. Verification / debugging tips
9. Cleanup

---

## 1. What Is NetworkPolicy?

- NetworkPolicy is the Kubernetes/OpenShift resource that defines **which pods, which namespaces and which ports** can reach each other (and the outside world).
- **Default behavior:** if a namespace has no NetworkPolicy, all pods can reach each other (and the outside) freely (allow-all).
- The moment at least one NetworkPolicy **selects** a pod (via `podSelector`), **only the traffic allowed by those policies** gets through for that pod — everything else becomes **default deny**. So "adding a restriction" really means "building an allow list".
- If a namespace has several NetworkPolicies, they combine **additively (union / OR)** — none narrows another, each only adds permissions.
- There are two directions: `Ingress` (traffic into the pod) and `Egress` (traffic out of the pod).

> **Important:** for NetworkPolicy to work, the cluster's SDN/CNI plugin must support it. The default on OpenShift 4.x, **OVNKubernetes**, fully supports it. Check before the demo:

```bash
oc get network.config/cluster -o jsonpath='{.status.networkType}'
# Expected output: OVNKubernetes
```

---

## 2. Setting Up the Demo Environment

Create 4 namespaces ("projects" in OpenShift) and give each a `team` label — `namespaceSelector` will use it.

```bash
for ns in poc-np-frontend poc-np-backend poc-np-other poc-np-monitoring; do
  oc new-project "$ns"
done

oc label namespace poc-np-frontend   team=frontend   --overwrite
oc label namespace poc-np-backend    team=backend    --overwrite
oc label namespace poc-np-other      team=other      --overwrite
oc label namespace poc-np-monitoring team=monitoring --overwrite
```

Install the target service in `poc-np-backend`. **Note:** the standard `nginx` image crashes under OpenShift's restricted SCC (random UID) because it cannot write to `/var/cache/nginx`; s2i builder images (`ubi9/nginx-124`) are not directly runnable servers either, they just print instructions. For a demo/test the least troublesome option is a simple `busybox httpd`, which runs fine with an arbitrary UID:

```bash
oc -n poc-np-backend create deployment web --image=busybox:1.36 \
  -- sh -c "mkdir -p /tmp/www && echo 'backend-ok' > /tmp/www/index.html && httpd -f -p 8080 -h /tmp/www"

oc -n poc-np-backend expose deployment web --port=8080

oc -n poc-np-backend get pods,svc
```

Start a "client" pod in each test namespace:

```bash
for ns in poc-np-frontend poc-np-other poc-np-monitoring; do
  oc -n $ns run tester --image=busybox:1.36 --restart=Never -- sleep 3600
done
```

**Baseline test (with no policy, all should succeed):**

```bash
for ns in poc-np-frontend poc-np-other poc-np-monitoring; do
  echo "--- $ns -> backend ---"
  oc -n $ns exec tester -- wget -qO- --timeout=3 web.poc-np-backend.svc.cluster.local:8080
done
```

✅ **Actual test output — all three got through:**

```
--- poc-np-frontend -> backend ---
backend-ok
--- poc-np-other -> backend ---
backend-ok
--- poc-np-monitoring -> backend ---
backend-ok
```

---

## 3. Scenario 1: Block All Traffic (Default Deny)

Block all ingress traffic to all pods in the `poc-np-backend` namespace.

```yaml
# deny-all-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: poc-np-backend
spec:
  podSelector: {}      # selects ALL pods in the namespace
  policyTypes:
    - Ingress
  # no ingress rule is defined, so no traffic is allowed
```

```bash
oc apply -f deny-all-ingress.yaml
```

**Test — all three should fail:**

```bash
for ns in poc-np-frontend poc-np-other poc-np-monitoring; do
  echo "--- $ns -> backend ---"
  oc -n $ns exec tester -- wget -qO- --timeout=3 web.poc-np-backend.svc.cluster.local:8080
done
```

✅ **Actual test output:**

```
--- poc-np-frontend -> backend ---
wget: download timed out
--- poc-np-other -> backend ---
wget: download timed out
--- poc-np-monitoring -> backend ---
wget: download timed out
```

Point to stress: `podSelector: {}` + an empty `ingress` field → traffic **from every direction** is rejected for all pods in that namespace.

---

## 4. Scenario 2: Allow Traffic Only From a Specific Namespace

On top of deny-all, add a policy that allows traffic only from the `frontend` namespace.

```yaml
# allow-from-frontend-namespace.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-frontend-namespace
  namespace: poc-np-backend
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              team: frontend
      ports:
        - protocol: TCP
          port: 8080
```

```bash
oc apply -f allow-from-frontend-namespace.yaml
```

✅ **Actual test output:**

```
--- poc-np-frontend -> backend ---
backend-ok
--- poc-np-other -> backend ---
wget: download timed out
--- poc-np-monitoring -> backend ---
wget: download timed out
```

> **Note:** `namespaceSelector` is label-based, not name-based. OpenShift automatically adds the `kubernetes.io/metadata.name` label to every namespace, which can be used to match by name:
> ```yaml
> - namespaceSelector:
>     matchLabels:
>       kubernetes.io/metadata.name: poc-np-frontend
> ```

### 4.1 Showing the additive behavior live: add the 4th namespace

Add a **second** allow policy to the same namespace to also allow `monitoring` — without changing the existing policy:

```yaml
# allow-from-monitoring-namespace.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-monitoring-namespace
  namespace: poc-np-backend
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              team: monitoring
      ports:
        - protocol: TCP
          port: 8080
```

```bash
oc apply -f allow-from-monitoring-namespace.yaml
oc -n poc-np-backend get networkpolicy
```

✅ **Actual test output (final state of the 4 namespaces):**

```
--- poc-np-frontend -> backend ---
backend-ok
--- poc-np-other -> backend ---
wget: download timed out
--- poc-np-monitoring -> backend ---
backend-ok
```

```
NAME                              POD-SELECTOR   AGE
allow-from-frontend-namespace     <none>         ...
allow-from-monitoring-namespace   <none>         ...
default-deny-ingress              <none>         ...
```

This is the clearest live proof of "NetworkPolicies don't override each other, they combine (OR)": `frontend` and `monitoring` get in, `other` is still out.

---

## 5. Scenario 3: Allow Traffic Only From a Specific Pod (Label)

Namespace-based permission is sometimes too broad. To allow only pods with specific labels, even within the same namespace, use `podSelector`. When `namespaceSelector` and `podSelector` are used together in the **same** `from` item, they work with **AND** logic.

First remove the broad permission given to the `frontend` namespace and replace it with a narrower policy that allows only a pod with a specific label:

```bash
oc -n poc-np-backend delete networkpolicy allow-from-frontend-namespace
```

```yaml
# allow-from-frontend-app-only.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-frontend-app-only
  namespace: poc-np-backend
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              team: frontend
          podSelector:
            matchLabels:
              role: api-client   # only pods with this label INSIDE the frontend namespace
      ports:
        - protocol: TCP
          port: 8080
```

```bash
oc apply -f allow-from-frontend-app-only.yaml
```

**Test — without the label, access should be rejected:**

```bash
oc -n poc-np-frontend exec tester -- wget -qO- --timeout=3 web.poc-np-backend.svc.cluster.local:8080
```

✅ `wget: download timed out` (expected)

**Now add the right label to the pod and test again:**

```bash
oc -n poc-np-frontend label pod tester role=api-client
oc -n poc-np-frontend exec tester -- wget -qO- --timeout=3 web.poc-np-backend.svc.cluster.local:8080
```

✅ **Actual test output:** `backend-ok` — access opened the instant the label was added (a very effective moment in a live demo).

> **Watch out (common mistake):** if you write two separate list items under `from` (`- namespaceSelector...` and, on a separate line, `- podSelector...`), it means **OR**. For AND, both must be in the **same** list item (as above, under the same `-`).

---

## 6. Scenario 4: Restricting Egress Traffic (+ a real pitfall)

Allow pods in the `frontend` namespace to reach only `backend` (port 8080) and DNS, and block everything else (e.g. the internet).

> **Correction note:** an early draft of this section claimed that writing `to: []` (an empty target list) means "allow no targets". That was **wrong** — as confirmed by the official API docs (`oc explain networkpolicy.spec.egress.to`) and an isolated test (taking the wrong DNS port out of the picture and trying a raw TCP handshake by IP with only `to: []` + the right port), **an empty or omitted `to` field means "allow every target"** (it restricts by port only, not by target). The real reason DNS broke in the first test was the one real pitfall below: the wrong port number.

### The real pitfall: Service port ≠ Pod port

On OpenShift the DNS namespace is `openshift-dns`, and the service appears to listen on port 53 — but the actual pod (CoreDNS) listens on **5353**; the Service just does a 53→5353 DNAT. **The `ports` field of a NetworkPolicy matches the target pod's actual (target/container) port, not the Service port.** If you write `port: 53`, the NetworkPolicy never matches the real pod and DNS breaks silently.

We confirmed the real port like this (`v1 Endpoints` gives a deprecation warning on 4.22; EndpointSlice is preferred):

```bash
oc get endpointslice -n openshift-dns -l kubernetes.io/service-name=dns-default -o jsonpath='{.items[0].ports}'
# ✅ Actual output: dns (UDP) 5353, dns-tcp (TCP) 5353, metrics (TCP) 9154
```

✅ **Actual output — with the same policy written with `port: 53`:**

```
nslookup web.poc-np-backend.svc.cluster.local   ->  ;; connection timed out; no servers could be reached
wget web.poc-np-backend.svc.cluster.local:8080  ->  wget: bad address 'web.poc-np-backend.svc.cluster.local:8080'
```

Even though backend is allowed, the app breaks silently because it cannot resolve DNS.

### The correct (tested, working) egress policy

```yaml
# restrict-egress-frontend.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-egress
  namespace: poc-np-frontend
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              team: backend
      ports:
        - protocol: TCP
          port: 8080
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: openshift-dns
      ports:
        - protocol: UDP
          port: 5353
        - protocol: TCP
          port: 5353
```

```bash
oc apply -f restrict-egress-frontend.yaml
```

✅ **Actual test output:**

```bash
oc -n poc-np-frontend exec tester -- nslookup web.poc-np-backend.svc.cluster.local
# Server:  172.30.0.10   Address: 172.30.0.10:53
# Name:    web.poc-np-backend.svc.cluster.local
# Address: 172.30.85.177          <-- DNS resolution works

oc -n poc-np-frontend exec tester -- wget -qO- --timeout=3 web.poc-np-backend.svc.cluster.local:8080
# backend-ok                       <-- backend access works

oc -n poc-np-frontend exec tester -- wget -qO- --timeout=3 http://1.1.1.1
# timeout                          <-- internet egress blocked (expected)
```

> **Demo note:** showing this pitfall (service port vs pod port) live and then fixing it drives home the message "if you don't know exactly what you're testing, NetworkPolicy will break you silently". On many Kubernetes clusters `kube-dns` port 53 = pod port 53, so this pitfall does not appear; it is an OpenShift-specific detail — mention that vanilla Kubernetes may behave differently.
>
> **Correct facts about the `to`/`from` field:** `to: []`, leaving it empty, or omitting the field entirely — all three mean the same thing: **"no restriction on target/source, filter by port only."** To restrict, add at least one `namespaceSelector`/`podSelector`/`ipBlock` to the list (we confirmed this with the `oc explain networkpolicy.spec.egress.to` output and a live isolated test).

---

## 7. Verification / Debugging Tips

```bash
# List all policies in the namespace
oc get networkpolicy -n poc-np-backend

# Details and selector info
oc describe networkpolicy default-deny-ingress -n poc-np-backend

# Confirm the cluster's SDN type (OVNKubernetes supports NetworkPolicy)
oc get network.config/cluster -o jsonpath='{.status.networkType}'

# Find a service's REAL target port (critical when writing egress policies)
oc get endpointslice -n <namespace> -l kubernetes.io/service-name=<service-name> -o jsonpath='{.items[0].ports}'

# Check namespace labels (for namespaceSelector mistakes)
oc get ns <namespace> --show-labels
```

---

## 8. Cleanup

```bash
oc delete project poc-np-frontend poc-np-backend poc-np-other poc-np-monitoring
```

---

## Summary

| Scenario | policyTypes | Key field |
|---|---|---|
| Block all traffic | Ingress | `podSelector: {}`, empty `ingress` |
| Allow a specific namespace | Ingress | `ingress.from.namespaceSelector` |
| Add a second namespace additively | Ingress | A 2nd NetworkPolicy object in the same ns (OR) |
| Allow a specific pod | Ingress | `ingress.from.podSelector` (+ namespaceSelector = AND, in the same list item) |
| Restrict egress | Egress | `egress.to` — must contain at least one selector to restrict |

**Golden rules:**
1. The moment the first NetworkPolicy selecting a pod is created, default-deny kicks in for that direction (ingress/egress); every policy added afterwards only defines **extra permissions** (allow/OR) and never tightens the restriction.
2. Leaving the `to`/`from` field as an empty array (`[]`) **and** omitting it entirely mean the same thing: **"no restriction on target/source"** (only `ports` remains as a filter). To restrict the target/source you must list at least one `namespaceSelector`/`podSelector`/`ipBlock`. (Source: `oc explain networkpolicy.spec.egress.to`, confirmed with a live isolated test.)
3. The `ports` field always matches **the target pod's real container port**, not the port the Service exposes — for services that do port mapping, such as DNS, confirm the real port with `oc get endpointslice`.
