# 13 — East-West Traffic Control Demo Guide (Restriction + Observability)

> 🇹🇷 [Türkçe](README.tr.md) · [← 12 — Logging + LokiStack](../12-logging/README.md) · [POC flow](../../README.md) · [14 — MultiCluster Observability →](../14-multicluster-observability/README.md)

This document shows how **service-to-service (east-west/lateral) traffic inside the cluster** is both **restricted with NetworkPolicy** and **observed through real log records** of those decisions (allow/deny) with **OVN ACL Logging**. The [08 — NetworkPolicy](../../C-network-and-security/08-network-policy/README.md) and [09 — Micro-Segmentation](../../C-network-and-security/09-micro-segmentation/README.md) guides only proved the **restriction** (connection went through / didn't); here we also show live that **every connection decision (allow/drop) — by whom, because of which policy, for which IP/port —** is visible as a raw log line.

Every step was **tested live on a lab cluster (OpenShift 4.22, **OVNKubernetes**) in the `poc-eastwest-demo` namespace** — real OVN ACL audit log lines were matched one-to-one against real pod IPs. If [12 — Logging](../12-logging/README.md) is installed, the same records can also be queried centrally from Loki (section 5.1).

Scenario order:

1. Concept — east-west traffic and why it needs to be observed
2. Setting up the demo environment (allowed source, source to block, protected target)
3. Restricting with NetworkPolicy + enabling ACL logging
4. Generating traffic and verifying (connection level)
5. **Observability**: real OVN ACL audit log records
6. A real pitfall: writing the annotation in the right place
7. Cleanup

---

## 1. Concept

- **North-south traffic**: traffic coming into the cluster from outside (Route/Ingress/LoadBalancer) or leaving it.
- **East-west traffic**: lateral traffic **between pods inside the cluster** (service to service, same or different namespace). In modern microservice architectures most traffic lives here — and the classic "perimeter firewall" approach (protect only the outer boundary) does not see it.
- **Restriction** (NetworkPolicy — see [08 — NetworkPolicy](../../C-network-and-security/08-network-policy/README.md), [09 — Micro-Segmentation](../../C-network-and-security/09-micro-segmentation/README.md)): defines which pod can reach which pod.
- **Observability** (this document): the audit trail that proves the restriction **really works**, **who tried**, and **when they were rejected**. On OVNKubernetes this comes from the **ACL Logging** feature — every OVN ACL rule generated from a NetworkPolicy (`allow`/`drop`) gets a log level, and a log line is produced on the node for every matching packet.

---

## 2. Setting Up the Demo Environment

Three pods: **allowed source** (`app-a`), **source to block** (`app-c`), **protected target** (`app-b`).

```yaml
# eastwest-demo.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: poc-eastwest-demo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-a-allowed
  namespace: poc-eastwest-demo
  labels: { role: allowed-source }
spec:
  replicas: 1
  selector: { matchLabels: { role: allowed-source } }
  template:
    metadata: { labels: { role: allowed-source } }
    spec:
      containers:
        - name: web
          image: busybox:1.36
          command: ["sh", "-c", "sleep 3600"]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-c-blocked
  namespace: poc-eastwest-demo
  labels: { role: blocked-source }
spec:
  replicas: 1
  selector: { matchLabels: { role: blocked-source } }
  template:
    metadata: { labels: { role: blocked-source } }
    spec:
      containers:
        - name: web
          image: busybox:1.36
          command: ["sh", "-c", "sleep 3600"]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-b-protected
  namespace: poc-eastwest-demo
  labels: { role: protected-target }
spec:
  replicas: 1
  selector: { matchLabels: { role: protected-target } }
  template:
    metadata: { labels: { role: protected-target } }
    spec:
      containers:
        - name: web
          image: busybox:1.36
          command: ["sh", "-c", "mkdir -p /tmp/www && echo APP-B-OK > /tmp/www/index.html && httpd -f -p 8080 -h /tmp/www"]
          ports: [{ containerPort: 8080 }]
---
apiVersion: v1
kind: Service
metadata: { name: app-b, namespace: poc-eastwest-demo }
spec: { selector: { role: protected-target }, ports: [{ port: 8080 }] }
```

```bash
oc apply -f eastwest-demo.yaml
```

---

## 3. Restricting With NetworkPolicy + ACL Logging

```yaml
# allow-only-app-a.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-only-app-a
  namespace: poc-eastwest-demo
spec:
  podSelector:
    matchLabels: { role: protected-target }
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector:
            matchLabels: { role: allowed-source }
      ports:
        - protocol: TCP
          port: 8080
```

```bash
oc apply -f allow-only-app-a.yaml
```

Enable ACL logging at the **namespace** level (see section 6 — critical detail: the annotation goes on the **Namespace**, not on the NetworkPolicy):

```bash
oc annotate namespace poc-eastwest-demo \
  'k8s.ovn.org/acl-logging={"deny":"alert","allow":"notice"}' --overwrite
```

- `deny` level: log severity for rejected connections (`alert`, `warning`, `notice`, `info`, `debug`).
- `allow` level: log severity for allowed connections.
- **All** NetworkPolicies and EgressFirewall rules in the namespace are affected by this setting.

---

## 4. Generating Traffic and Verifying (Connection Level)

```bash
A_POD=$(oc get pod -n poc-eastwest-demo -l role=allowed-source -o jsonpath='{.items[0].metadata.name}')
C_POD=$(oc get pod -n poc-eastwest-demo -l role=blocked-source -o jsonpath='{.items[0].metadata.name}')

echo "app-a -> app-b (should be allowed):"
oc exec "$A_POD" -n poc-eastwest-demo -- wget -qO- --timeout=3 app-b.poc-eastwest-demo.svc.cluster.local:8080

echo "app-c -> app-b (should be blocked):"
oc exec "$C_POD" -n poc-eastwest-demo -- wget -qO- --timeout=3 app-b.poc-eastwest-demo.svc.cluster.local:8080
```

✅ **Actual output:**

```
app-a -> app-b (should be allowed):
APP-B-OK

app-c -> app-b (should be blocked):
wget: download timed out
command terminated with exit code 1
```

Up to here it is the same as the [08 — NetworkPolicy](../../C-network-and-security/08-network-policy/README.md)/[09 — Micro-Segmentation](../../C-network-and-security/09-micro-segmentation/README.md) demos — the restriction works. The real difference is in the next section.

---

## 5. Observability: Real OVN ACL Audit Log Records

Confirm on the node that the decisions really are **logged** (on the node where `app-b` runs):

```bash
NODE=$(oc get pod -n poc-eastwest-demo -l role=protected-target -o jsonpath='{.items[0].spec.nodeName}')
oc debug node/$NODE --to-namespace=default -- chroot /host grep poc-eastwest-demo /var/log/ovn/acl-audit-log.log | tail -n 20
```

✅ **Actual output** (matches the pod IPs one-to-one — `app-a=...96`, `app-c=...97`, `app-b=...98`; same result in a second test round with `.179/.180/.181`):

```
...|INFO|name="NP:poc-eastwest-demo:allow-only-app-a:Ingress:0", verdict=allow, severity=notice, direction=to-lport:
    tcp,...,nw_src=10.131.3.96,nw_dst=10.131.3.98,...,tp_src=39036,tp_dst=8080,tcp_flags=syn
...|INFO|name="NP:poc-eastwest-demo:allow-only-app-a:Ingress:0", verdict=allow, severity=notice, direction=to-lport:
    tcp,...,nw_src=10.131.3.96,nw_dst=10.131.3.98,...,tcp_flags=ack
...|INFO|name="NP:poc-eastwest-demo:allow-only-app-a:Ingress:0", verdict=allow, severity=notice, direction=to-lport:
    tcp,...,nw_src=10.131.3.96,nw_dst=10.131.3.98,...,tcp_flags=psh|ack
...
...|INFO|name="NP:poc-eastwest-demo:Ingress", verdict=drop, severity=alert, direction=to-lport:
    tcp,...,nw_src=10.131.3.97,nw_dst=10.131.3.98,...,tp_src=60386,tp_dst=8080,tcp_flags=syn
...|INFO|name="NP:poc-eastwest-demo:Ingress", verdict=drop, severity=alert, direction=to-lport:
    tcp,...,nw_src=10.131.3.97,nw_dst=10.131.3.98,...,tcp_flags=syn
```

**How to read it:**

- First group (`verdict=allow`, `severity=notice`, ACL name `NP:poc-eastwest-demo:allow-only-app-a:Ingress:0`): **the whole** TCP handshake of `app-a` (`.96`) to `app-b` (`.98`) (syn → ack → psh|ack → ack → fin|ack) — shown together with the name of the NetworkPolicy that allowed it.
- Second group (`verdict=drop`, `severity=alert`, ACL name `NP:poc-eastwest-demo:Ingress` — the general/implicit default-deny): **every SYN packet** from `app-c` (`.97`) trying to reach `app-b` **is rejected and logged one by one** (the client's repeated attempts show up as TCP retransmissions).

**This fully meets the "observable" requirement:** we don't just say "the connection failed"; we can read from the raw log line **who (source IP), since when, and which policy** blocked it.

### 5.1 Central observability: querying ACL records from Loki

OVN ACL audit logs are part of the **`audit`** log type in OpenShift Logging (not `infrastructure`). Because the `ClusterLogForwarder` from [12 — Logging](../12-logging/README.md) already collects the `audit` input, they flow to Loki **with no extra configuration**:

```bash
TOKEN=$(oc create token log-reader -n openshift-logging --duration=1h)
ROUTE=$(oc get route logging-loki -n openshift-logging -o jsonpath='https://{.spec.host}')
curl -sk -H "Authorization: Bearer $TOKEN" \
  --data-urlencode 'query={log_type="audit"} |= "NP:poc-eastwest-demo"' \
  --data-urlencode "start=$(($(date +%s)-1800))000000000" --data-urlencode "end=$(date +%s)000000000" \
  "$ROUTE/api/logs/v1/audit/loki/api/v1/query_range"
```

✅ **Actual output:** **7 records** for this namespace (5 `verdict=allow`, 2 `verdict=drop`); the same as the file on the node. Stream labels: `log_type=audit`, `k8s_node_name=<node>`. These records have **no** `log_source` label; to pick out ACL records use text filters such as `|= "NP:<namespace>"` or `|= "verdict=drop"`. This way, rejected east-west traffic across the whole cluster can be tracked with a single query, and alerts can be built on top, without `oc debug`-ing into each node.

---

## 6. A Real Pitfall: Writing the Annotation in the Right Place

On the first try we wrote the `k8s.ovn.org/acl-logging` annotation on the **NetworkPolicy** object — it **silently had no effect**: the restriction worked (`app-c` was blocked), but the log file on the node had **0** lines for this namespace (✅ same result in a second test round; no error message). The right way: the annotation must be written on the **Namespace** object:

```bash
# WRONG (no effect, and no error either):
oc annotate networkpolicy allow-only-app-a -n poc-eastwest-demo 'k8s.ovn.org/acl-logging=...'

# RIGHT:
oc annotate namespace poc-eastwest-demo 'k8s.ovn.org/acl-logging=...'
```

Because it gives no warning at all, this pitfall is hard to spot — don't assume "logging is on" without confirming the file is actually filling up (`wc -l`).

---

## 7. Cleanup

```bash
oc delete project poc-eastwest-demo
```

---

## Summary

| Component | Role |
|---|---|
| `NetworkPolicy` | **Restricts** east-west traffic (allow/deny decision) |
| `Namespace` annotation `k8s.ovn.org/acl-logging` | Turns on **logging** of all NetworkPolicy/EgressFirewall decisions in that namespace |
| `/var/log/ovn/acl-audit-log.log` (on the node) | Raw record of every allow/drop decision — source/target IP, port, TCP flag, which policy |
| `ClusterLogForwarder` **audit** input | Ships these logs to central Loki (no extra configuration) — see [12 — Logging](../12-logging/README.md) |

**Golden rules:**
1. The ACL logging annotation goes on the **Namespace**, not the NetworkPolicy — writing it in the wrong place silently does nothing.
2. `deny`/`allow` severities are set **separately** — usually `deny: alert` (unexpected rejected traffic that should draw attention) + `allow: notice`/`info` (lower priority for normal, high-volume traffic) is a sensible default.
3. The logs are on the node's local file system (`/var/log/ovn/acl-audit-log.log`); the **`audit`** input of the ClusterLogForwarder in [12 — Logging](../12-logging/README.md) ships them to Loki automatically.
4. Restriction (NetworkPolicy) and observability (ACL logging) are **two independent** mechanisms that are switched on and off separately — one does not bring the other automatically.
