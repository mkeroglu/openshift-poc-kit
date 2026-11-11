# 14 — MultiClusterObservability With ODF (S3) on OpenShift — End-to-End CLI Guide

> 🇹🇷 [Türkçe](README.tr.md) · [← 13 — East-West Traffic Control](../13-east-west-traffic-control/README.md) · [POC flow](../../README.md) · [15 — ODF Performance Test →](../15-odf-performance-test/README.md)

This document walks through installing **MultiClusterObservability (MCO)**, a component of Red Hat Advanced Cluster Management (ACM), entirely with the `oc` CLI, using a bucket created through the S3-compatible **NooBaa** Multi-Cloud Gateway of **OpenShift Data Foundation (ODF)** for metric storage (Thanos object storage).

> Every step was **tested end to end on a lab cluster** (OpenShift 4.22, ACM hub + 2 managed clusters); the "✅ Actual output" lines come from that test.

## Prerequisites

- You must be logged in with `oc` as a cluster administrator (cluster-admin).
- **ACM / MultiClusterHub** must be installed and `Running`:

  ```bash
  oc get multiclusterhub -A
  ```

- The **ODF** operator must be installed and a `StorageCluster` must exist:

  ```bash
  oc get csv -n openshift-storage | grep odf-operator
  oc get storagecluster -n openshift-storage
  ```

- Confirm the MCO CRDs exist on the cluster (they ship with the ACM observability component):

  ```bash
  oc get crd multiclusterobservabilities.observability.open-cluster-management.io
  ```

## Step 1 — Create an S3 bucket on ODF/NooBaa (ObjectBucketClaim)

ODF fulfills S3 bucket requests with `ObjectBucketClaim` (OBC), a Kubernetes-native resource. The `openshift-storage.noobaa.io` storage class creates the bucket automatically using NooBaa's own S3 service.

```bash
cat <<'EOF' | oc apply -f -
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  name: multiclusterobs
  namespace: openshift-storage
spec:
  generateBucketName: multiclusterobs
  storageClassName: openshift-storage.noobaa.io
  additionalConfig:
    bucketclass: noobaa-default-bucket-class
EOF
```

Wait for the OBC to be `Bound`:

```bash
oc wait --for=jsonpath='{.status.phase}'=Bound \
  obc/multiclusterobs -n openshift-storage --timeout=120s
```

Once the OBC is bound, ODF automatically creates a **Secret** (access keys) and a **ConfigMap** (bucket/endpoint info) with the same name:

```bash
oc get obc multiclusterobs -n openshift-storage
oc get secret multiclusterobs -n openshift-storage
oc get cm multiclusterobs -n openshift-storage -o yaml
```

Example ConfigMap content:

```yaml
data:
  BUCKET_HOST: s3.openshift-storage.svc
  BUCKET_NAME: multiclusterobs-<uuid>
  BUCKET_PORT: "443"
```

## Step 2 — Read the bucket details and credentials

```bash
BUCKET_HOST=$(oc get cm multiclusterobs -n openshift-storage -o jsonpath='{.data.BUCKET_HOST}')
BUCKET_NAME=$(oc get cm multiclusterobs -n openshift-storage -o jsonpath='{.data.BUCKET_NAME}')
ACCESS_KEY=$(oc get secret multiclusterobs -n openshift-storage -o jsonpath='{.data.AWS_ACCESS_KEY_ID}' | base64 -d)
SECRET_KEY=$(oc get secret multiclusterobs -n openshift-storage -o jsonpath='{.data.AWS_SECRET_ACCESS_KEY}' | base64 -d)
```

Also confirm the address of the in-cluster NooBaa S3 service (LoadBalancer/ClusterIP service, port 80/443):

```bash
oc get svc s3 -n openshift-storage
```

## Step 3 — Observability namespace and the Thanos object-storage secret

MCO's components run in the `open-cluster-management-observability` namespace; it should already exist from the ACM/MCH install (if not, create it with `oc create ns open-cluster-management-observability`).

Create a Secret containing `thanos.yaml`, which tells Thanos how to connect to S3:

```bash
cat <<EOF > /tmp/thanos.yaml
type: s3
config:
  bucket: ${BUCKET_NAME}
  endpoint: ${BUCKET_HOST}:80
  insecure: true
  access_key: ${ACCESS_KEY}
  secret_key: ${SECRET_KEY}
EOF

# If the secret already exists (from a previous install), "create" fails; this line creates it if missing and updates it if present:
oc create secret generic thanos-object-storage \
  -n open-cluster-management-observability \
  --from-file=thanos.yaml=/tmp/thanos.yaml --dry-run=client -o yaml | oc apply -f -

rm -f /tmp/thanos.yaml
```

> ⚠️ **Leftover secret:** `thanos-object-storage` may be left over from an earlier MCO install; if it points to a bucket that no longer exists, Thanos cannot write data. In the lab test, a months-old secret whose OBC had been deleted was found; it was backed up first and then updated with the new bucket details. Check: `oc get secret thanos-object-storage -n open-cluster-management-observability -o jsonpath='{.metadata.creationTimestamp}'` and `oc get obc -A`.

> **Note:** the `endpoint` field uses port **80** and `insecure: true` because the NooBaa S3 service is reached over plain HTTP from inside the cluster (the traffic does not leave the cluster network). For cross-cluster / external access, prefer TLS (443, `insecure: false`).

## Step 4 — Choose the StorageClass

MCO's Thanos components (Alertmanager, Compact, Receive, Rule, Store) need a block storage StorageClass for their own PVCs. ODF's default RBD class is usually suitable:

```bash
oc get storageclass
# e.g. ocs-storagecluster-ceph-rbd (default)
```

## Step 5 — Create the MultiClusterObservability CR

```bash
cat <<'EOF' | oc apply -f -
apiVersion: observability.open-cluster-management.io/v1beta2
kind: MultiClusterObservability
metadata:
  name: observability
spec:
  observabilityAddonSpec: {}
  storageConfig:
    metricObjectStorage:
      name: thanos-object-storage
      key: thanos.yaml
    storageClass: ocs-storagecluster-ceph-rbd
    alertmanagerStorageSize: 1Gi
    compactStorageSize: 100Gi
    receiveStorageSize: 25Gi
    ruleStorageSize: 1Gi
    storeStorageSize: 25Gi
EOF
```

Field notes:

| Field | Description |
|---|---|
| `metricObjectStorage.name/key` | Name of the Secret created in step 3 and the key inside it |
| `storageClass` | Block storage class to use for the PVCs |
| `*StorageSize` | PVC size of each Thanos component (adjust as needed) |

In the `v1beta2` API, `observabilityAddonSpec` and `storageConfig` are required fields; schema validation can be checked through the CRD:

```bash
oc get crd multiclusterobservabilities.observability.open-cluster-management.io -o json \
  | jq '.spec.versions[] | select(.storage==true) | .schema.openAPIV3Schema.properties.spec.required'
```

## Step 6 — Verify the installation

```bash
# CR status
oc get multiclusterobservability observability -o jsonpath='{range .status.conditions[*]}{.type}={.status} ({.reason}): {.message}{"\n"}{end}'

# Pods coming up
oc get pods -n open-cluster-management-observability -w
```

When the installation is done you should see the `Installing=False`, `Ready=True` conditions, and these components should be `Running`/`Ready` in the namespace:

- `observability-alertmanager-*`
- `observability-grafana-*`
- `observability-observatorium-api-*`
- `observability-observatorium-operator-*`
- `observability-rbac-query-proxy-*`
- `observability-thanos-compact-*`
- `observability-thanos-query-*` / `query-frontend-*`
- `observability-thanos-receive-*`
- `observability-thanos-rule-*`
- `observability-thanos-store-shard-*`
- `observability-thanos-store-memcached-*`
- `endpoint-observability-operator-*`

Route for Grafana access:

```bash
oc get route -n open-cluster-management-observability
```

✅ **Actual output:** MCO `Ready=True` in **123 s**. Within a few minutes all the components above were `Running` (thanos-receive ×3, store-shard ×3, query/query-frontend, observatorium-api ×2, grafana ×2, alertmanager ×3, metrics-collector). Routes: `alertmanager`, `grafana`, `observatorium-api`, `rbac-query-proxy`.

**The addon on managed clusters:**

```bash
oc get managedclusteraddon observability-controller -A
```

✅ **Actual output:** `Available=True (ManagedClusterAddOnLeaseUpdated)` on the reachable managed cluster. `Unknown` on the cluster that was already unreachable at the time (`ManagedCluster available=Unknown`). There is no separate addon for the hub cluster; the hub's metrics are collected by `metrics-collector-deployment`.

**Confirm that metrics are really flowing** (pods being `Running` is not enough):

```bash
oc port-forward -n open-cluster-management-observability svc/observability-thanos-query-frontend 19090:9090 &
curl -s --get --data-urlencode 'query=count by (cluster) (up)' http://127.0.0.1:19090/api/v1/query
```

✅ **Actual output (~5 min after install):**

| cluster | `count(up)` | `count({__name__=~"cluster:.*"})` |
|---|---|---|
| hub | 357 | 523 |
| managed cluster | 130 | 516 |

Right after the install, the collector log showed one `unable to forward results ... observatorium-api` error (the API was not ready yet); subsequent pushes were `metrics pushed successfully`.

## Known behaviors / common situations

- **`observability-thanos-store-shard-*` pods may stay `0/1 Running` for a long time.** If the bucket already holds many Thanos blocks (e.g. MCO was installed and removed before without deleting the bucket data), the store gateway loads each block's index header one by one. This is normal, not an error; follow progress with:

  ```bash
  oc logs observability-thanos-store-shard-0-0 \
    -n open-cluster-management-observability -f | grep "loaded new block"
  ```

  If it is slow but the `thanos_blocks_meta_sync_failures_total` metric is `0` (check from inside the pod with `curl localhost:10902/metrics`), there is no real problem; it just takes time.

- A **`FailedToRetrieveImagePullSecret: multiclusterhub-operator-pull-secret`** warning may appear in the events. If the images are already cached on the node (`Pulled ... already present on machine`), this is not blocking; if an image pull fails when scheduling onto a new node, make sure ACM/MCH's global pull secret is synced to that namespace.

- **To reuse a bucket**: if an OBC/Secret created earlier is already `Bound` (`oc get obc -A`), you can skip creating a new bucket and go straight to step 3 (create the `thanos-object-storage` secret with the existing bucket details).

## Cleaning up resources (optional, cannot be undone)

```bash
oc delete multiclusterobservability observability
oc delete pvc --all -n open-cluster-management-observability   # 13 PVCs (~250Gi) are left behind after MCO is deleted
oc delete secret thanos-object-storage -n open-cluster-management-observability
oc delete obc multiclusterobs -n openshift-storage   # also deletes the bucket data
```

✅ **Actual output:** when the MCO CR was deleted, the pods and the `observability-controller` addons on the managed clusters were removed automatically, but **13 PVCs** of Thanos/Alertmanager **were left**; they had to be deleted by hand.

> Deleting the `obc` **permanently deletes all data** in the NooBaa bucket. Deleting only the MCO CR keeps the bucket and the metric data in it.
