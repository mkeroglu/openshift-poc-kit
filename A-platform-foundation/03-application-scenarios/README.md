# 03 — Application Scenarios

> 🇹🇷 [Türkçe](README.tr.md) · [← 02 — Container Management](../02-container-management/README.md) · [POC flow](../../README.md) · [04 — Image Management →](../04-image-management/README.md)

This section shows how different application characteristics (stateless, stateful, API/microservice) are modeled on OpenShift. All examples run in the `ocp-poc` namespace (see `../02-container-management/namespace.yaml`). Every step was **tested end to end on a live lab cluster** (OpenShift 4.22).

> **Prerequisite:** the ResourceQuota/LimitRange created in 02 must be removed before this guide (see [02 — Container Management](../02-container-management/README.md) → "Before Moving to the Next Guide"). Otherwise `postgres-0` cannot be created because of the quota.

- [x] Stateless web app
- [x] Stateful database app
- [x] API/microservice app

---

## Stateless Web App

### What does stateless mean?

An application is **stateless** when it does not keep any persistent state about the requests it handles inside itself (on its disk or in memory). In practice this means:

- Any replica/pod can replace any other one exactly (**interchangeable**) — it does not matter which pod serves the request.
- Deleting and recreating a pod (crash, node drain, rolling update) **causes no data loss**, because no persistent data was kept in the first place.
- No stable network identity (ordered hostnames like `nginx-0`, `nginx-1`) is needed; the Service load-balances to the pods round-robin.
- Scaling up/down is unordered and immediate — there is no "first 0, then 1" ordering like in a StatefulSet.

In OpenShift/Kubernetes, the resource that provides this behavior natively is the **Deployment** (the opposite of the **StatefulSet** covered in the next section).

### Example: nginx (from the Container Management guide)

Instead of building a separate example, we reuse the `nginx` Deployment created in **[02 — Container Management](../02-container-management/README.md)** — it is a classic stateless web app: 2 replicas, serving a static `index.html` from a `ConfigMap`, never producing or changing data on its own disk. (ConfigMap/Secret volumes are read-only and identical in every pod, so they do not count as "state" — they are not data the application produces at runtime.)

If it is not installed yet:

```bash
oc apply -f ../02-container-management/namespace.yaml
oc apply -f ../02-container-management/nginx.yaml
```

### A scenario that proves statelessness

**1) Note the current pods:**

```bash
oc get pods -l app=nginx -o wide
```

**2) Delete one of the pods at random** — unlike a StatefulSet, it does not matter "which" pod is deleted:

```bash
POD=$(oc get pods -l app=nginx -o jsonpath='{.items[0].metadata.name}')
oc delete pod "$POD"
```

**3) Verify the result** — the ReplicaSet controller creates a new pod with a different name/IP, but:

- The Service (`ClusterIP`) and Route do not change; the client notices nothing.
- The page content is exactly the same — because the data lived in the ConfigMap, not on the pod's own disk.
- No "data recovery" or "synchronization" step was needed.

```bash
oc get pods -l app=nginx -o wide            # new pod name/IP differs, behavior is the same
curl -s "http://$(oc get route nginx -o jsonpath='{.spec.host}')"   # content unchanged
```

**4) Scaling is unordered too** — it does not matter which pod is "first" (there is no ordered identity like `nginx-0`, `nginx-1` in a StatefulSet; pod names always have a random suffix):

```bash
oc scale deployment/nginx --replicas=4
oc get pods -l app=nginx -o wide
oc scale deployment/nginx --replicas=2
```

✅ **Actual output:** while the pod was being deleted, **20 out of 20 requests** sent through the Route half a second apart returned `200`. The deleted pod was replaced by a new one with a different name and IP (`...-5rdrv/10.131.0.208` → `...-9pc7r/10.131.0.209`), and the page content did not change. Scaling to 4 replicas created four pods with random suffixes.

### Summary: Stateless (Deployment) vs Stateful (StatefulSet)

| | Deployment (stateless) | StatefulSet (stateful) |
|---|---|---|
| Pod name | Random suffix (`nginx-6b65f9-28w78`) | Ordered, stable (`postgres-0`, `postgres-1`) |
| When a pod is deleted | Any order, no data loss | Comes back with the same name/identity, reattaching to its own PVC |
| Scaling order | Irrelevant, parallel | Ordered (0 → 1 → 2) |
| Storage | Usually none / shared read-only | Each pod has **its own** PVC |

The next section (**Stateful database app**) shows this comparison with a concrete example.

## Stateful Database App

### What does stateful mean?

Unlike a Deployment, a **StatefulSet** gives each pod a **persistent identity** and **its own persistent storage** (PersistentVolumeClaim):

- Pod names are ordered and stable (`postgres-0`, `postgres-1`, ...) — even after being deleted and recreated, a pod comes back **with the same name**.
- Each pod has **its own** PVC through `volumeClaimTemplates` (e.g. `data-postgres-0`); deleting the pod does not delete this PVC (or its data), and when the new pod comes back with the same name it reattaches to **the same PVC**.
- Pod creation/deletion is **ordered** (created 0 → 1 → 2, deleted in reverse) — not parallel/unordered like a Deployment.
- A **headless Service** (`clusterIP: None`) is usually used for a stable network identity, so each pod is reachable at a fixed DNS name like `postgres-0.postgres.ocp-poc.svc.cluster.local`.

### Example: PostgreSQL

What's in `postgres.yaml`:

- **Secret (`postgres-credentials`)** — `POSTGRESQL_USER`, `POSTGRESQL_PASSWORD`, `POSTGRESQL_DATABASE`.
- **Service (`postgres`)** — `clusterIP: None` (headless), matching the StatefulSet's `serviceName` field.
- **StatefulSet (`postgres`)** — `quay.io/sclorg/postgresql-16-c9s` image, 1 replica, a 1Gi persistent disk via `volumeClaimTemplates`.

> **Image note:** `quay.io/sclorg/postgresql-16-c9s` comes from Red Hat's "Software Collections" (sclorg) project and is **designed to run with OpenShift's random-UID model** (group permissions set to `g=u`). Unlike the nginx/php-apache sections, it **needs no extra SCC** (`anyuid`, etc.) — an example of how choosing the right image can avoid these problems entirely.

### Creating with `oc apply`

```bash
oc project ocp-poc
oc apply -f postgres.yaml
oc get pods -l app=postgres -w
oc get pvc
```

### Imperative equivalent

```bash
oc create secret generic postgres-credentials \
  --from-literal=POSTGRESQL_USER=appuser \
  --from-literal=POSTGRESQL_PASSWORD=app-pass-2026 \
  --from-literal=POSTGRESQL_DATABASE=appdb

# headless service (--clusterip=None is required for the StatefulSet's DNS identity)
oc create service clusterip postgres --tcp=5432:5432 --clusterip=None
```

> **No imperative equivalent:** `oc create`/`kubectl create` has shortcuts for Deployment, Job and CronJob, but **there is no imperative subcommand for StatefulSet** (✅ live test: `oc create statefulset` → `error: Unexpected args`; the secret and `clusterIP=None` headless service commands above worked fine). StatefulSets can only be created from a YAML manifest with `oc apply -f` (or `oc create -f`).

### Scenario: data persistence and pod identity

**1) Create a sample table and data:**

```bash
oc exec -it postgres-0 -- psql -U appuser -d appdb -c \
  "CREATE TABLE notes (id serial PRIMARY KEY, message text, created_at timestamptz DEFAULT now());"
oc exec -it postgres-0 -- psql -U appuser -d appdb -c \
  "INSERT INTO notes (message) VALUES ('Hello OpenShift POC'), ('OpenShift');"
oc exec -it postgres-0 -- psql -U appuser -d appdb -c "SELECT * FROM notes;"
```

**2) Delete the pod and verify that identity and data are kept** — the exact opposite of the stateless scenario, the pod comes back **with the same name**:

```bash
oc delete pod postgres-0
oc get pods -l app=postgres -w      # the new pod is also created as "postgres-0"
oc exec -it postgres-0 -- psql -U appuser -d appdb -c "SELECT * FROM notes;"   # data is still there
```

✅ **Actual output:** **19 s** after deletion, the new pod was `Ready` again as `postgres-0` and attached to the same `data-postgres-0` PVC. The query returned the two rows from before the deletion (`1 | Hello OpenShift POC`, `2 | OpenShift`). The pod ran with the `restricted-v2` SCC; no extra SCC was needed.

**3) Observe the ordered nature of scaling:**

```bash
oc scale statefulset/postgres --replicas=2
oc get pods -l app=postgres -o wide     # postgres-1 is created, with ITS OWN PVC (data-postgres-1)
oc get pvc
oc scale statefulset/postgres --replicas=1
oc get pvc   # even though postgres-1 is gone, the data-postgres-1 PVC is STILL THERE (default behavior)
```

✅ **Actual output:** `replicas=2` → `postgres-0`, `postgres-1` and `data-postgres-0`, `data-postgres-1` (both `Bound`). `replicas=1` → only `postgres-0` was left, but the `data-postgres-1` PVC stayed `Bound`.

> **Important:** scaling down or deleting the StatefulSet entirely (`oc delete -f postgres.yaml`) **does not delete** its PVCs (a deliberate design to prevent data loss). To clean up unused PVCs you must delete them by hand: `oc delete pvc -l app=postgres`.

### Deleting

```bash
oc delete -f postgres.yaml    # deletes StatefulSet + Service + Secret, PVCs STAY
oc delete pvc -l app=postgres # extra step if you want to delete the data permanently
```

In the GUI: **Administrator** perspective → **Workloads > StatefulSets** → `⋮` → **Delete StatefulSet** (PVCs must also be deleted separately under **Storage > PersistentVolumeClaims**).

## API/Microservice App

### What does this scenario show?

This section builds a realistic scenario that combines the previous two: **PostgREST** — a lightweight service that automatically generates a REST API from a PostgreSQL schema, needs no extra code or build, and is configured entirely through environment variables. This way:

- **The API/microservice layer is itself stateless** (Deployment, 2 replicas, any pod can be deleted/scaled) — but now it **does real work**: it connects to the `postgres` StatefulSet from the **Stateful Database App** section and serves real data.
- Configuration comes entirely from **environment variables** (ConfigMap + Secret) — no files are mounted, a classic "12-factor" microservice practice.
- The service's own **health endpoint** (`/`) also reflects the health of the database connection — if it cannot reach the DB, `/` fails too, which gives a realistic signal for readiness/liveness probes.

> This shows that "stateless" does not mean "does not handle data" — it only means it **does not keep data persistently on its own disk/memory**; the data lives in a separate, persistent layer (StatefulSet + PVC).

### Prerequisite

`postgres.yaml` from the **Stateful Database App** section must be applied and the `notes` table/data created (see step 1 of the scenario above).

### What's in `postgrest.yaml`

- **ConfigMap (`postgrest-config`)** — non-sensitive settings: `PGRST_DB_SCHEMA`, `PGRST_DB_ANON_ROLE`, `PGRST_SERVER_PORT`.
- **Secret (`postgrest-db-uri`)** — the connection string (in a Secret because it contains the username/password): `PGRST_DB_URI`.
- **Deployment (`postgrest`)** — 2 replicas, `postgrest/postgrest` image, port 3000.
- **Service** and **Route (`postgrest`)** — for external access.

> For simplicity, this POC uses the connection owner `appuser` directly as `PGRST_DB_ANON_ROLE`. In production, define a separate, restricted read-only role with `SELECT` on only the tables it needs.

### Creating with `oc apply`

```bash
oc apply -f postgrest.yaml
oc get pods -l app=postgrest -w
```

### Test: reading data through the API

```bash
HOST=$(oc get route postgrest -o jsonpath='{.spec.host}')
curl -s "http://$HOST/"          # root endpoint: OpenAPI schema (200 means the DB connection is healthy)
curl -s "http://$HOST/notes"     # real data from the postgres StatefulSet, as JSON
```

✅ **Actual output:** `GET /` → `200`. `GET /notes` → `[{"id":1,"message":"Hello OpenShift POC",...}, {"id":2,"message":"OpenShift",...}]`.

### Scenario: stateless resilience of the API layer

```bash
POD=$(oc get pods -l app=postgrest -o jsonpath='{.items[0].metadata.name}')
oc delete pod "$POD"
curl -s "http://$HOST/notes"     # no interruption, the Service routes to the other replica
oc scale deployment/postgrest --replicas=4
oc scale deployment/postgrest --replicas=2
```

✅ **Actual output:** while a PostgREST pod was being deleted, **20 out of 20 requests** returned `200`; scaling to 4 replicas completed without issues.

This behavior is exactly the same as the nginx scenario in **Stateless Web App** — the difference is that this time there is a real, persistent data source (StatefulSet) behind the API.

### Deleting

```bash
oc delete -f postgrest.yaml
```

In the GUI: in the **Developer** perspective click the `postgrest` Deployment in **Topology** and choose **Actions > Delete Deployment**, or delete it from **Workloads > Deployments** in the **Administrator** perspective.
