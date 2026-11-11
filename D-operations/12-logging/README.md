# 12 — OpenShift Logging + LokiStack (ODF/S3) Setup Guide

> 🇹🇷 [Türkçe](README.tr.md) · [← 11 — Security Tests](../../C-network-and-security/11-security-tests/README.md) · [POC flow](../../README.md) · [13 — East-West Traffic Control →](../13-east-west-traffic-control/README.md)

This document installs **OpenShift Logging (Cluster Logging Operator)** + **Loki Operator (LokiStack)** and forwards **application** and **audit** logs to Loki. For storage (S3), instead of a separate external S3, the S3-compatible object storage of **ODF (OpenShift Data Foundation)**, already installed on the cluster, is used. Every step was **tested end to end on a lab cluster** (OpenShift 4.22, Logging 6.6) — a test pod wrote logs in the `ocp-poc` namespace, and the LokiStack API confirmed that both application and audit tenants really reached Loki.

Scenario order:

1. Prerequisite check (can the Loki Operator / Cluster Logging Operator be installed, is ODF healthy)
2. Installing the operators
3. Preparing S3 object storage for LokiStack (ODF)
4. Creating the LokiStack
5. Forwarding application + audit logs to Loki with a ClusterLogForwarder
6. Verification (actual test output)
7. **Real pitfalls** (3 real problems hit and solved on the live environment)
8. Cleanup

---

## 1. Prerequisite Check

**a) Are the required operators in the catalog?**

```bash
oc get packagemanifest -n openshift-marketplace | grep -iE "^loki-operator|^cluster-logging"
```

✅ Both are in the **Red Hat Operators** catalog, with `stable-6.6` as the default channel.

**The operators may already be installed** (by another team or a previous effort): `oc get csv -A | grep -E 'loki-operator|cluster-logging' | grep -v Copied`. If they are, skip section 2. In the lab test both were already installed as `v6.6.1 Succeeded`.

**b) Is ODF healthy for the S3 backend?**

Two options were considered for LokiStack's object storage — ODF has **two different S3 engines**, and although both were installed on this cluster, **only one was healthy**:

```bash
oc get noobaa -n openshift-storage
oc get cephobjectstore -n openshift-storage
```

- **NooBaa (MCG)** — `Available: False` (`INVALID_SCHEMA_REPLY SERVER system_api#/methods/read_system`) in the first test round; `Ready` in the next round. NooBaa's state can change over time.
- **Ceph RGW (`CephObjectStore`)** — `PHASE: Ready` in both rounds.

**Result:** this guide uses **Ceph RGW** (the `ocs-storagecluster-ceph-rgw` storage class) as the S3 backend. In your environment, check both first and use the healthy one.

---

## 2. Installing the Operators

```bash
cat <<'EOF' | oc apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-operators-redhat
  labels:
    openshift.io/cluster-monitoring: "true"
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-operators-redhat
  namespace: openshift-operators-redhat
spec: {}
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: loki-operator
  namespace: openshift-operators-redhat
spec:
  channel: stable-6.6
  name: loki-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  installPlanApproval: Automatic
---
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-logging
  labels:
    openshift.io/cluster-monitoring: "true"
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-logging
  namespace: openshift-logging
spec:
  targetNamespaces:
    - openshift-logging
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: cluster-logging
  namespace: openshift-logging
spec:
  channel: stable-6.6
  name: cluster-logging
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  installPlanApproval: Automatic
EOF
```

Verify the installation:

```bash
oc get csv -n openshift-operators-redhat
oc get csv -n openshift-logging
```

✅ **Actual output (~1 minute later):** both operators `PHASE: Succeeded`.

---

## 3. S3 Object Storage for LokiStack (ODF/Ceph RGW)

Create an S3-compatible bucket on ODF with an `ObjectBucketClaim` (OBC):

```yaml
# loki-bucket-obc.yaml
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  name: loki-bucket-odf
  namespace: openshift-logging
spec:
  generateBucketName: loki-bucket-odf
  storageClassName: ocs-storagecluster-ceph-rgw   # use RGW if NooBaa is unhealthy
```

```bash
oc apply -f loki-bucket-obc.yaml
oc get obc loki-bucket-odf -n openshift-logging
```

✅ **Actual output:** `PHASE: Bound` (within seconds — when tried with NooBaa it stayed `Pending` for hours, see section 7).

The OBC creates a ConfigMap (bucket name/host/port) and a Secret (access/secret key). From them, create a new secret in the format LokiStack expects:

```bash
ACCESS_KEY=$(oc get secret loki-bucket-odf -n openshift-logging -o jsonpath='{.data.AWS_ACCESS_KEY_ID}' | base64 -d)
SECRET_KEY=$(oc get secret loki-bucket-odf -n openshift-logging -o jsonpath='{.data.AWS_SECRET_ACCESS_KEY}' | base64 -d)
BUCKET=$(oc get cm loki-bucket-odf -n openshift-logging -o jsonpath='{.data.BUCKET_NAME}')

oc create secret generic logging-loki-s3 -n openshift-logging \
  --from-literal=access_key_id="$ACCESS_KEY" \
  --from-literal=access_key_secret="$SECRET_KEY" \
  --from-literal=bucketnames="$BUCKET" \
  --from-literal=endpoint="http://rook-ceph-rgw-ocs-storagecluster-cephobjectstore.openshift-storage.svc:80" \
  --from-literal=region="us-east-1"
```

> ⚠️ **Secret left over from a previous install:** if `logging-loki-s3` already exists, `oc create secret` fails with `already exists` and **LokiStack runs with the old (probably deleted) bucket's details**. In the live test a secret left over from an earlier effort was found. In that case, replace the secret and restart the Loki components:
>
> ```bash
> oc create secret generic logging-loki-s3 -n openshift-logging ... --dry-run=client -o yaml | oc replace -f -
> oc rollout restart deploy,sts -n openshift-logging -l app.kubernetes.io/instance=logging-loki
> ```

> **Note:** the endpoint uses **HTTP (port 80)**, not HTTPS (443). RGW's certificate is signed by Ceph's own internal CA; rather than giving LokiStack a separate CA bundle, HTTP is enough and simpler for a POC because this traffic is cluster-internal anyway (ClusterIP, not reachable from outside). In production prefer HTTPS with a CA bundle.

---

## 4. Creating the LokiStack

```yaml
# lokistack.yaml
apiVersion: loki.grafana.com/v1
kind: LokiStack
metadata:
  name: logging-loki
  namespace: openshift-logging
spec:
  size: 1x.demo              # POC/demo size — see section 7 (may be too small for production)
  storage:
    schemas:
      - version: v13
        effectiveDate: "2024-01-01"
    secret:
      name: logging-loki-s3
      type: s3
  storageClassName: ocs-storagecluster-ceph-rbd   # RWO block storage for WAL/index
  tenants:
    mode: openshift-logging   # ready-made multi-tenant mode using OpenShift's own RBAC/auth
  limits:
    global:
      ingestion:
        ingestionRate: 40        # MB/s — the 1x.demo default is 4; audit logs alone exceed it (see 7.3)
        ingestionBurstSize: 60
```

```bash
oc apply -f lokistack.yaml
oc get pods -n openshift-logging -l app.kubernetes.io/instance=logging-loki
oc get lokistack logging-loki -n openshift-logging -o jsonpath='{.status.conditions}'
```

✅ **Actual output (~50–90 seconds later):** compactor, distributor, gateway (x2), index-gateway, ingester, querier, query-frontend — all `Running`, with `type: Ready, status: "True", reason: ReadyComponents` in `status.conditions`.

---

## 5. ClusterLogForwarder: Application + Audit → Loki

**a) ServiceAccount + RBAC for the collector** (permission to collect application + audit only — infrastructure logs are deliberately left out):

```bash
cat <<'EOF' | oc apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: collector
  namespace: openshift-logging
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: collector-application-logs
subjects:
  - kind: ServiceAccount
    name: collector
    namespace: openshift-logging
roleRef:
  kind: ClusterRole
  name: collect-application-logs
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: collector-audit-logs
subjects:
  - kind: ServiceAccount
    name: collector
    namespace: openshift-logging
roleRef:
  kind: ClusterRole
  name: collect-audit-logs
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: collector-logs-writer
subjects:
  - kind: ServiceAccount
    name: collector
    namespace: openshift-logging
roleRef:
  kind: ClusterRole
  name: logging-collector-logs-writer
  apiGroup: rbac.authorization.k8s.io
EOF
```

**b) ClusterLogForwarder:**

```yaml
# clusterlogforwarder.yaml
apiVersion: observability.openshift.io/v1
kind: ClusterLogForwarder
metadata:
  name: collector
  namespace: openshift-logging
spec:
  serviceAccount:
    name: collector
  collector:
    resources:
      requests:
        cpu: 200m
        memory: 1Gi
      limits:
        memory: 4Gi          # see section 7 — the default limit caused OOMKilled on this cluster
  inputs:
    - name: app-logs
      type: application
      application: {}        # ALL namespaces — narrowed with includes in the POC test, see section 7.3
      # application:
      #   includes:
      #     - namespace: ocp-poc
    - name: audit-logs
      type: audit
      audit: {}
  outputs:
    - name: loki-output
      type: lokiStack
      lokiStack:
        target:
          name: logging-loki
          namespace: openshift-logging
        authentication:
          token:
            from: serviceAccount
      tls:
        ca:
          key: service-ca.crt
          configMapName: openshift-service-ca.crt
  pipelines:
    - name: app-and-audit-to-loki
      inputRefs:
        - app-logs
        - audit-logs
      outputRefs:
        - loki-output
```

```bash
oc apply -f clusterlogforwarder.yaml
oc get pods -n openshift-logging -l app.kubernetes.io/component=collector
```

✅ **Actual output:** `status.conditions`: `Authorized=True`, `Valid=True`, `Ready=True`. One `collector-xxxxx` pod (DaemonSet) on each (running) node, `1/1 Running`, no restarts. Collector memory usage measured between 468Mi and 1240Mi (limit 4Gi).

---

## 6. Verification (actual test output)

**a) Produce logs for the test:**

```bash
oc run log-test-emitter -n ocp-poc --image=busybox:1.36 --restart=Never -- \
  sh -c 'for i in $(seq 1 20); do echo "LOKI_DEMO_MARKER_line_$i $(date)"; sleep 1; done; sleep 180'
```

**b) A token with read permission to query the Loki gateway:**

```bash
cat <<'EOF' | oc apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: loki-log-reader
rules:
  - apiGroups: [loki.grafana.com]
    resources: [application, audit]
    resourceNames: [logs]
    verbs: [get]
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: log-reader
  namespace: openshift-logging
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: log-reader-binding
subjects:
  - kind: ServiceAccount
    name: log-reader
    namespace: openshift-logging
roleRef:
  kind: ClusterRole
  name: loki-log-reader
  apiGroup: rbac.authorization.k8s.io
EOF

TOKEN=$(oc create token log-reader -n openshift-logging --duration=1h)
ROUTE=$(oc get route logging-loki -n openshift-logging -o jsonpath='https://{.spec.host}')
```

**c) Query the application tenant:**

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  --data-urlencode 'query={k8s_namespace_name="ocp-poc"} |= "LOKI_DEMO_MARKER"' \
  --data-urlencode "start=$(($(date +%s)-300))000000000" \
  --data-urlencode "end=$(date +%s)000000000" \
  "$ROUTE/api/logs/v1/application/loki/api/v1/query_range"
```

✅ **Actual output:** `"status":"success"`, stream `k8s_namespace_name=ocp-poc, k8s_pod_name=log-test-emitter`, with **all 20** `LOKI_DEMO_MARKER_line_1..20` lines in `values`, as full JSON log records (with `kubernetes.*`, `@timestamp`, `message` fields).

**d) Query the audit tenant:**

```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  --data-urlencode 'query={log_type="audit"}' \
  --data-urlencode "start=$(($(date +%s)-600))000000000" \
  --data-urlencode "end=$(date +%s)000000000" \
  --data-urlencode 'limit=3' \
  "$ROUTE/api/logs/v1/audit/loki/api/v1/query_range"
```

✅ **Actual output:** `"status":"success"`, real kube-apiserver audit records from the control plane nodes (e.g. `verb: get`, `resource: olmconfigs`, with fields like `annotations."authorization.k8s.io/decision":"allow"`, `"authorization.k8s.io/reason":"RBAC: allowed by ClusterRoleBinding ..."`).

**Result: both application and audit logs reach Loki end to end and can be queried through the gateway.**

---

## 7. Real Pitfalls (hit on the live environment)

### 7.1 Pick the healthy S3 engine, not necessarily NooBaa

ODF can have **two separate S3-compatible backends**: NooBaa (MCG) and Ceph RGW. On this cluster NooBaa had been unhealthy for days with an `INVALID_SCHEMA_REPLY` error; an OBC created with `storageClassName: openshift-storage.noobaa.io` stayed **`Pending` indefinitely**. Check both before installing and pick **the healthy one** (section 1b).

### 7.2 The collector's default memory limit is too low on busy clusters

This cluster hosts ~90 namespaces. When the ClusterLogForwarder was created without `spec.collector.resources`, the collector pods (vector) kept getting **`OOMKilled`** (the default limit is very low). Fix: raise `spec.collector.resources.limits.memory` (raised to `4Gi` in this POC, after which it stayed stable).

### 7.3 The `1x.demo` size is too small for real cluster-wide log volume (including audit)

The LokiStack `1x.demo` size (single replica, for demos) has a default **`ingestionRate: 4MB/s`** limit. When the `application` input was left covering **all namespaces** (`application: {}`, no filter), the cluster's real total log volume far exceeded that limit — the distributor rejected **all** application writes:

```
level=error ... msg="write operation failed" details="ingestion rate limit exceeded for user
application (limit: 4194304 bytes/sec) ..." org_id=application
```

**Audit logs hit the same limit.** In the second test round, even with the application input narrowed to a single namespace, the distributor errored **449 times** in 5 minutes, all from the audit tenant:

```
details="ingestion rate limit exceeded for user audit (limit: 4194304 bytes/sec) while attempting to ingest '1461' lines totaling '2517323' bytes ..."
```

Kube-apiserver audit log volume depends on the cluster's size and API traffic; on a mid-sized cluster it can exceed 4 MB/s on its own. That is why the limit was added to `lokistack.yaml` by default.

Raising `spec.limits.global.ingestion.ingestionRate` (to `40` MB/s in this POC) stopped the errors (✅ in the live test, **0** errors in 2 minutes after the patch), but a single-replica `1x.demo` is still not guaranteed to handle real production volume. **Two practical options:**

- **In production:** pick the LokiStack size based on real log volume (`1x.small`/`1x.medium`/`1x.large` — see the ingestion-rate table in the Red Hat documentation), and use `1x.demo` only for real demos/tests.
- **To narrow the scope:** collect only the namespaces you care about with `spec.inputs[].application.includes` (this is what the POC's live test did — `includes: [{namespace: ocp-poc}]` — it did not hit the ingestion limit and did not pull other tenants' log volume into Loki unnecessarily):

```yaml
inputs:
  - name: app-logs
    type: application
    application:
      includes:
        - namespace: ocp-poc
```

---

## 8. Cleanup

```bash
oc delete clusterlogforwarder collector -n openshift-logging
oc delete lokistack logging-loki -n openshift-logging
oc delete pvc -n openshift-logging -l app.kubernetes.io/instance=logging-loki   # PVCs stay when the LokiStack is deleted
oc delete secret logging-loki-s3 -n openshift-logging
oc delete obc loki-bucket-odf -n openshift-logging
oc delete clusterrolebinding collector-application-logs collector-audit-logs collector-logs-writer log-reader-binding
oc delete clusterrole loki-log-reader
oc delete sa collector log-reader -n openshift-logging
oc delete subscription loki-operator -n openshift-operators-redhat
oc delete subscription cluster-logging -n openshift-logging
oc delete ns openshift-logging openshift-operators-redhat
```

---

## Summary

| Component | Namespace | Role |
|---|---|---|
| Loki Operator | `openshift-operators-redhat` | Manages the LokiStack CRD |
| Cluster Logging Operator | `openshift-logging` | Manages the ClusterLogForwarder CRD |
| `ObjectBucketClaim` (Ceph RGW) | `openshift-logging` | LokiStack's S3 backend (chunk storage) |
| `LokiStack` (`logging-loki`) | `openshift-logging` | Loki itself (distributor/ingester/querier/gateway, etc.) |
| `collector` ServiceAccount | `openshift-logging` | Identity of the Vector collector pods — allowed to collect app+audit only |
| `ClusterLogForwarder` (`collector`) | `openshift-logging` | Which logs (application, audit) go where (LokiStack) |

**Golden rules:**
1. Before installing, check whether the operators and the `logging-loki-s3` secret **already exist**; an old secret connects LokiStack to the wrong bucket.
2. If ODF has more than one S3 backend (NooBaa + RGW), pick **the healthy one** before installing — with an unhealthy NooBaa the OBC stays `Pending` forever without any error message.
3. On busy clusters, **always** give the collector an explicit memory limit with `spec.collector.resources` — the default limit can cause OOMKilled.
4. A `1x.demo` LokiStack **may not handle real production log volume — including audit logs** — either increase the size or narrow the scope with `application.includes`; otherwise the distributor silently loses data (yours included).
5. Verify by sending a real query to the LokiStack gateway API (`/api/logs/v1/<tenant>/loki/api/v1/query_range`) and by searching the distributor logs for `rate limit exceeded` — pods being `Running` does not mean data is actually reaching Loki.
