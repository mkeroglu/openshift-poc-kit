# 02 — Container Management

> 🇹🇷 [Türkçe](README.tr.md) · [← 01 — Identity and Access](../01-identity-and-access/README.md) · [POC flow](../../README.md) · [03 — Application Scenarios →](../03-application-scenarios/README.md)

This section covers container/pod management on OpenShift. Every step was **tested end to end on a live lab cluster** (OpenShift 4.22); the "✅ Actual output" lines come from those tests.

- [x] Namespace/Project structure
- [x] Deployment rollout/rollback
- [x] Horizontal Pod Autoscaler
- [x] Resource Quota
- [x] LimitRange

---

## Namespace/Project Structure

### What is a namespace?

A namespace is a partitioning mechanism that logically isolates resources (Pods, Services, Deployments, ConfigMaps, etc.) inside a Kubernetes/OpenShift cluster. It lets different teams, environments (dev/test/prod) or applications share one cluster without mixing their resources or clashing on names. Resource quotas (ResourceQuota), authorization (RBAC) and network policies (NetworkPolicy) are usually defined per namespace.

### What is a Project?

A Project is an OpenShift-specific resource (`project.openshift.io/v1`) that extends the Kubernetes Namespace. Creating a Project automatically creates a Namespace with the same name. On top of the Namespace, a Project adds automatic assignment of the default RBAC roles (admin, edit, view), extra metadata such as `display-name` and `description`, and a dedicated view in the OpenShift Web Console.

**In short:** every Project is a Namespace, but not every Namespace is a Project. A plain Namespace created with `oc apply -f namespace.yaml` may not get the same rich defaults as one created with `oc new-project`; however, if the `openshift.io/display-name` and `openshift.io/description` annotations are added, the Console shows it like a Project.

### YAML file

`namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ocp-poc
  labels:
    app.kubernetes.io/part-of: ocp-poc
    environment: poc
  annotations:
    openshift.io/display-name: "OCP POC"
    openshift.io/description: "Namespace/project for the OpenShift POC"
```

### How to apply the YAML

```bash
# Create or update the Namespace/Project
oc apply -f namespace.yaml

# Verify
oc get namespace ocp-poc
oc get project ocp-poc
oc describe project ocp-poc
```

`oc apply` is declarative: the definition in the YAML is compared with the current state in the cluster; the resource is created if missing and patched by the difference if it exists. So it is safe to run repeatedly (idempotent).

### Creating a project with `oc`, without apply

To create a new project directly from the command line, without a YAML file:

```bash
oc new-project ocp-poc \
  --display-name="OCP POC" \
  --description="Namespace/project for the OpenShift POC"
```

This command creates the Namespace/Project and also switches your current context (`oc project`) to the new project. To switch to an existing project:

```bash
oc project ocp-poc
```

### Creating a project from the GUI (Web Console)

1. Log in to the OpenShift Web Console.
2. Switch to the **Developer** perspective at the top left (or stay in **Administrator**).
3. Go to **Home > Projects** in the left menu.
4. Click **Create Project** at the top right.
5. In the form:
   - **Name**: `ocp-poc`
   - **Display Name**: `OCP POC`
   - **Description**: `Namespace/project for the OpenShift POC`
6. Click **Create**.
7. Afterwards you can switch to this project with the project selector at the top.

### Deleting a Namespace/Project

**With the CLI:**

```bash
# Delete as a Project (recommended, OpenShift-specific resource)
oc delete project ocp-poc

# or delete as a plain Namespace
oc delete namespace ocp-poc

# Delete via the YAML file
oc delete -f namespace.yaml
```

> Note: deletion cannot be undone, and **all** resources in the namespace (Pods, Deployments, Services, ConfigMaps, Secrets, etc.) are deleted with it. Use with care in production.

**With the GUI:**

1. Go to **Home > Projects** in the Web Console.
2. Click the `⋮` (kebab) menu on the row of the project to delete (`ocp-poc`).
3. Choose **Delete Project**.
4. Type the project name (`ocp-poc`) in the confirmation dialog to confirm.

---

## Deployment rollout/rollback

In this section, `nginx.yaml` installs a simple Nginx application made of a Deployment, ConfigMap, Secret, Service and Route into the `ocp-poc` namespace. Then a rollout/rollback scenario is shown on that Deployment with `oc rollout` commands.

### What's in nginx.yaml

- **Secret (`nginx-conf`)** — holds Nginx's `conf.d/default.conf` in `stringData`. A custom server block is defined because the container runs unprivileged on port 8080.
- **ConfigMap (`nginx-index-html`)** — holds the static `index.html` page.
- **Deployment (`nginx`)** — a pod template using the `nginxinc/nginx-unprivileged:1.27-alpine` image, 2 replicas, mounting the ConfigMap and Secret as volumes.
- **Service (`nginx`)** — exposes the Deployment inside the cluster on port 8080.
- **Route (`nginx`)** — exposes the Service outside the cluster over HTTP (no host is set, so OpenShift generates a hostname).

> **Why `nginx-unprivileged` and 8080?** The standard `nginx` image has trouble under OpenShift's default **restricted** SCC (random non-root UID, no capabilities): it cannot bind to a privileged port like 80 (no `NET_BIND_SERVICE` capability), and it cannot write to directories such as `/var/cache/nginx`. The `nginxinc/nginx-unprivileged` image is built for this case and listens on 8080, which is why this POC uses it.

The matching `index.html` and `nginx.conf` files are also included as plain files in the same directory (`index.html`, `nginx.conf`) for the `oc create ... --from-file` examples below.

### Creating with `oc apply`

```bash
oc project ocp-poc   # make sure we are in the right namespace
oc apply -f nginx.yaml
```

Verify:

```bash
oc get deployment,configmap,secret,svc,route -l app=nginx
oc get pods -l app=nginx -w
oc get route nginx -o jsonpath='{.spec.host}'
```

A `curl` to the Route hostname shows the `index.html` content and the custom `X-Served-By` header added in `nginx.conf`:

```bash
curl -I http://$(oc get route nginx -o jsonpath='{.spec.host}')
```

✅ **Actual output:** Deployment `2/2` ready; `curl -I` → `HTTP/1.1 200 OK`, `x-served-by: ocp-poc-nginx`; page body contains `<h1>OpenShift POC</h1>`. The same app, installed in a separate namespace with the imperative commands below, returned the same response.

### Creating the same resources imperatively (`oc create` / `oc expose`)

Instead of `nginx.yaml`, the same resources can be created one by one with imperative commands:

```bash
# ConfigMap - from the index.html file
oc create configmap nginx-index-html --from-file=index.html=index.html

# Secret - from the nginx.conf file
oc create secret generic nginx-conf --from-file=nginx.conf=nginx.conf

# Deployment - from the image
oc create deployment nginx --image=nginxinc/nginx-unprivileged:1.27-alpine --replicas=2

# Mount the ConfigMap into the Deployment as a volume
oc set volume deployment/nginx --add --name=index-html \
  --type=configmap --configmap-name=nginx-index-html \
  --mount-path=/usr/share/nginx/html/index.html --sub-path=index.html

# Mount the Secret into the Deployment as a volume
oc set volume deployment/nginx --add --name=nginx-conf \
  --type=secret --secret-name=nginx-conf \
  --mount-path=/etc/nginx/conf.d/default.conf --sub-path=nginx.conf

# Expose the Deployment as an in-cluster Service
oc expose deployment nginx --port=8080 --target-port=8080 --name=nginx

# Expose the Service outside the cluster with a Route
oc expose svc/nginx
```

> A Deployment created with `oc create deployment` does not include details such as readiness/liveness probes or resource limits that the YAML has. They can be added later with `oc set probe`, `oc set resources` or `oc edit deployment nginx`. Real projects prefer declarative YAML (GitOps); imperative commands are for quick prototyping and learning.

### Rollout / Rollback Scenario

`oc rollout` commands manage the revision history of a Deployment.

**1) View the current state and history**

```bash
oc rollout status deployment/nginx
oc rollout history deployment/nginx
```

**2) Publish a new (deliberately broken) version — trigger a rollout**

```bash
# simulate a "broken" deploy with an image tag that does not exist
oc set image deployment/nginx nginx=nginxinc/nginx-unprivileged:this-tag-does-not-exist
```

**3) Watch the rollout**

```bash
oc rollout status deployment/nginx
# new pods will go into ImagePullBackOff / ErrImagePull
oc get pods -l app=nginx
```

✅ **Actual output:** `oc rollout status` → `error: timed out waiting for the condition`. Pod list: two pods of the old ReplicaSet `Running`, the new pod `ImagePullBackOff`. The rolling update did not stop the old pods because the new pod never became ready; the app kept running without interruption.

**4) Roll back the broken rollout**

```bash
# go back to the previous working revision
oc rollout undo deployment/nginx

# to go back to a specific revision:
oc rollout history deployment/nginx                  # lists revision numbers
oc rollout history deployment/nginx --revision=2      # shows that revision's details
oc rollout undo deployment/nginx --to-revision=2
```

✅ **Actual output:** `deployment.apps/nginx rolled back` → `successfully rolled out`, image back to `nginxinc/nginx-unprivileged:1.27-alpine`.

**5) Ship several changes as one rollout: pause / resume**

```bash
oc rollout pause deployment/nginx
oc set image deployment/nginx nginx=nginxinc/nginx-unprivileged:1.27-alpine
oc set env deployment/nginx POC_VERSION=v2
oc rollout resume deployment/nginx   # changes made while paused are applied at once
```

✅ **Actual output:** changes made while paused did not reach the pods. After `resume`, a single rollout started the new pods with the `POC_VERSION=v2` environment variable.

> Resource limits (`oc set resources`) are **not changed** in this step: the Resource Quota calculations below assume nginx runs with the values from `nginx.yaml` (limits 200m/128Mi).

**6) Restart pods without changing the image**

```bash
oc rollout restart deployment/nginx
```

> **Important:** if you change `index.html` or `nginx.conf` and run `oc apply -f nginx.yaml` again, the Deployment's pod template (image, env, etc.) has not changed, so **no new rollout is triggered** — the existing pods keep running with the old ConfigMap/Secret content. Run `oc rollout restart deployment/nginx` for the new content to reach the pods.

## Horizontal Pod Autoscaler

### What is an HPA?

A HorizontalPodAutoscaler (HPA) is a Kubernetes/OpenShift resource that automatically increases or decreases the number of pods (replicas) of a workload such as a Deployment/StatefulSet based on observed metrics (CPU/memory usage by default, optionally custom/external metrics). The HPA controller measures the target metric periodically (every 15 seconds by default) and computes the desired replica count roughly as:

```
desiredReplicas = ceil( currentReplicas × ( currentMetricValue / targetMetricValue ) )
```

For a CPU-based HPA to work, the target container **must define `resources.requests.cpu`** — the utilization percentage is calculated against that request.

### Demo scenario: `php-apache`

Instead of trying to drive CPU with "natural" requests like in the nginx example (it is hard to load the CPU of a static web server meaningfully, and the result is unpredictable), we use the **`php-apache`** demo image from the official Kubernetes/OpenShift HPA docs. The image deliberately does a CPU-heavy calculation (a sqrt loop) for every request, so CPU rises clearly and predictably under load and the HPA's scale-up/scale-down behavior is easy to observe.

What's in `hpa.yaml`:

- **Deployment (`php-apache`)** — `registry.k8s.io/hpa-example` image, `requests.cpu: 200m` / `limits.cpu: 500m`.
- **Service (`php-apache`)** — ClusterIP for in-cluster access, port 80.
- **Route (`php-apache`)** — to test from a browser as well (optional, not needed for the load test).
- **HorizontalPodAutoscaler (`php-apache`)** — `minReplicas: 1`, `maxReplicas: 5`, target CPU utilization `50%`. To see the demo in a reasonable time, `behavior.scaleDown.stabilizationWindowSeconds` is lowered to **60 seconds** (the default is 300 — in production it is usually kept at the default or higher to prevent flapping on sudden drops).

> **SCC / port 80 note:** `hpa-example` is a classic Apache/root image and cannot bind to port 80 under OpenShift's default **restricted** SCC (random UID, no `NET_BIND_SERVICE` capability).
>
> ✅ **Actual output:** with the `restricted-v2` SCC the pod went into `Error`: `(13)Permission denied: AH00072: make_sock: could not bind to address 0.0.0.0:80`, `no listening sockets available, shutting down`.
>
> To fix it, grant the `anyuid` SCC to this namespace's `default` service account (requires cluster-admin). After that the pod ran with `scc=anyuid`:
>
> ```bash
> oc adm policy add-scc-to-user anyuid -z default -n ocp-poc
> ```
>
> This is for demo/POC only; real projects should rebuild the image to be OpenShift-compatible (ready for an arbitrary UID) — covered later in the **[04 — Image Management](../04-image-management/README.md)** and **[05 — CI/CD](../../B-application-delivery/05-ci-cd/README.md)** guides.

### Creating with `oc apply`

```bash
oc project ocp-poc
oc apply -f hpa.yaml

# if needed (see the SCC note above):
oc adm policy add-scc-to-user anyuid -z default -n ocp-poc
oc rollout restart deployment/php-apache
```

Verify:

```bash
oc get deployment,svc,route,hpa -l app=php-apache
oc describe hpa php-apache
```

### Imperative equivalent: `oc autoscale`

When the deployment already exists, the HPA in `hpa.yaml` can also be created with a single command:

```bash
oc create deployment php-apache --image=registry.k8s.io/hpa-example
oc set resources deployment/php-apache -c php-apache --requests=cpu=200m,memory=128Mi --limits=cpu=500m,memory=256Mi
oc expose deployment php-apache --port=80
oc expose svc/php-apache

oc autoscale deployment/php-apache --cpu-percent=50 --min=1 --max=5
```

### Creating an HPA from the GUI (Web Console)

1. In the **Developer** perspective go to **Topology** and click the `php-apache` Deployment.
2. In the side panel choose **Actions > Add HorizontalPodAutoscaler** (or in the **Administrator** perspective **Workloads > HorizontalPodAutoscalers > Create HorizontalPodAutoscaler**).
3. Enter **Minimum/Maximum Pods**: `1` / `5`, **CPU Utilization**: target `50%`.
4. Save with **Create**. The HPA is listed under **Workloads > HorizontalPodAutoscalers**, where current/target metric values can be watched.

### Scale-up / Scale-down Scenario

**1) Observe the initial state** (watch live in one terminal):

```bash
oc get hpa php-apache -w
```

Watch the pods in a second terminal:

```bash
oc get pods -l app=php-apache -w
```

**2) Generate load** — the Job in `load-generator.yaml` sends continuous requests to the `php-apache` service from 4 parallel pods:

```bash
oc apply -f load-generator.yaml
oc get pods -l app=load-generator
```

> Alternative (one-off, leaves no resources behind): `oc run -i --tty load-generator --rm --image=busybox:1.36 --restart=Never -- /bin/sh -c "while true; do wget -q -O- http://php-apache; done"`

**3) Watch the scale-up** — 4 parallel load-generator pods produce enough requests that within about **1 minute** CPU goes far above the `50%` target and the HPA quickly reaches `maxReplicas: 5`:

✅ **Actual output (`oc get hpa`, after load started):**

```
 31s  TARGETS=56%/50%   REPLICAS=1
 52s  TARGETS=249%/50%  REPLICAS=2
 62s  TARGETS=249%/50%  REPLICAS=5
124s  TARGETS=124%/50%  REPLICAS=5     # per-pod usage dropped once spread over 5 pods
```

```bash
oc describe hpa php-apache   # shows "SuccessfulRescale" messages under Events
```

**4) Stop the load** and watch the scale-down:

```bash
oc delete -f load-generator.yaml
```

After CPU drops, the HPA first waits for our `stabilizationWindowSeconds: 60`, then — because of the `Type: Pods, Value: 1, Period: 60s` rule in `behavior.scaleDown.policies` — removes **at most 1 pod every 60 seconds** until it reaches `minReplicas: 1`. Going all the way from `5` to `1` takes about **4–5 minutes** in total. Watch it live with `oc get hpa php-apache -w`.

✅ **Actual output (after the load was stopped):** CPU fell to `12%` at ~85 s and to `0%` at ~96 s. Replicas went **5 → 4 (138 s) → 3 (201 s) → 2 (264 s) → 1 (327 s)**, one pod per minute. `oc describe hpa` Events: `New size: 4; reason: All metrics below target` ... `New size: 1`.

### Deleting the HPA

**With the CLI:**

```bash
oc delete hpa php-apache
# or remove all demo resources together:
oc delete -f hpa.yaml
```

**With the GUI:**

1. In the **Administrator** perspective go to **Workloads > HorizontalPodAutoscalers**.
2. From the `⋮` menu on the `php-apache` row choose **Delete HorizontalPodAutoscaler** and confirm.

> Note: deleting the HPA does not delete the Deployment or its pods; it only stops autoscaling, and the replica count stays at its last value.

## Resource Quota

### What is a ResourceQuota?

A ResourceQuota caps the **total** resource consumption in a namespace (CPU, memory, number of pods/configmaps/secrets/services/PVCs, etc.). Even if the pod count grows through the HPA or manual scaling, the namespace **will not allow** a new pod that would exceed these limits — the Deployment/ReplicaSet tries to create the pod, but it is rejected at admission and stays `Pending`/never created.

> **Important:** if a ResourceQuota defines `requests.cpu`/`requests.memory`/`limits.cpu`/`limits.memory`, **every** pod in that namespace must now set those fields explicitly; otherwise pod creation is rejected with an error like "must specify limits.cpu". That is why a ResourceQuota is used together with the **LimitRange** below: the LimitRange hides this requirement from users by injecting default requests/limits into containers that do not set them.

What's in `resourcequota.yaml` — for the `ocp-poc` namespace:

| Field | Value | Description |
|---|---|---|
| `requests.cpu` | 500m | Total CPU **requests** of all pods in the namespace |
| `requests.memory` | 512Mi | Total memory **requests** of all pods in the namespace |
| `limits.cpu` | 1 | Total CPU **limits** of all pods |
| `limits.memory` | 1Gi | Total memory **limits** of all pods |
| `pods` | 10 | Maximum number of pods running at the same time |
| `configmaps`, `secrets`, `services`, `persistentvolumeclaims` | 20/20/10/5 | Upper limits on the count of each resource |

### Creating with `oc apply`

```bash
oc project ocp-poc
oc apply -f resourcequota.yaml
oc describe resourcequota ocp-poc-quota
```

If the `nginx` (2 replicas, requests 50m/64Mi, limits 200m/128Mi) and `php-apache` (1 replica, requests 200m/128Mi, limits 500m/256Mi) deployments from the previous sections are already running, `oc describe resourcequota` shows the following **Used** values — all still below the hard limits, so existing pods are not affected (✅ these exact values were seen in the live test):

```
Resource            Used   Hard
--------            ----   ----
requests.cpu        300m   500m
requests.memory     256Mi  512Mi
limits.cpu          900m   1
limits.memory       512Mi  1Gi
pods                3      10
```

### Imperative equivalent: `oc create quota`

```bash
oc create quota ocp-poc-quota \
  --hard=requests.cpu=500m,requests.memory=512Mi,limits.cpu=1,limits.memory=1Gi,pods=10
```

Count quotas (`configmaps`, `secrets`, `services`, `persistentvolumeclaims`) can be added to the same `--hard` list: `--hard=...,configmaps=20,secrets=20,services=10,persistentvolumeclaims=5` (verified in the live test). Still, YAML (GitOps) is preferred for permanent definitions.

### Scenario: exceeding the quota

In the table above `limits.cpu` is already at **900m/1000m** (90%) — each new nginx pod asks for 200m more `limits.cpu`, so scaling to just **3** replicas is enough to exceed the quota (900m + 200m = 1100m > 1000m hard):

```bash
oc scale deployment/nginx --replicas=3
oc get replicaset -l app=nginx
oc describe replicaset <new-nginx-replicaset-name>
```

The **Events** section of `describe` shows an error like:

```
Warning  FailedCreate  ...  Error creating: pods "nginx-xxxxxxxxxx-yyyyy" is forbidden:
exceeded quota: ocp-poc-quota, requested: limits.cpu=200m,
used: limits.cpu=900m, limited: limits.cpu=1
```

✅ **Actual output:** ReplicaSet `desired=3 current=2 ready=2`; Event: `pods "nginx-...-6gwgs" is forbidden: exceeded quota: ocp-poc-quota, requested: limits.cpu=200m, used: limits.cpu=900m, limited: limits.cpu=1`.

The Deployment keeps running on the healthy pods — the 3rd pod is never created, so there is no outage. Verify and revert:

```bash
oc describe resourcequota ocp-poc-quota   # shows Used did not change
oc scale deployment/nginx --replicas=2       # back to the previous state
```

### Deleting the ResourceQuota

```bash
oc delete resourcequota ocp-poc-quota
# or
oc delete -f resourcequota.yaml
```

In the GUI: **Administrator** perspective → **Administration > ResourceQuotas** → `⋮` menu on the row → **Delete ResourceQuota**.

## LimitRange

### What is a LimitRange?

Unlike a ResourceQuota, a LimitRange sets limits per **individual** Pod/Container/PVC: the minimum/maximum CPU and memory a container may get, and the **default** limit (`default`) and **default request** (`defaultRequest`) applied when none is set.

What's in `limitrange.yaml` — `ocp-poc-limits`:

| Scope | Field | Value |
|---|---|---|
| Container | `defaultRequest` (cpu/memory) | 50m / 64Mi |
| Container | `default` (cpu/memory limit) | 100m / 128Mi |
| Container | `min` | 25m / 32Mi |
| Container | `max` | 500m / 512Mi |
| Pod | `max` (total) | 1 cpu / 1Gi |
| PersistentVolumeClaim | `min` / `max` | 1Gi / 10Gi |

### Creating with `oc apply`

```bash
oc apply -f limitrange.yaml
oc describe limitrange ocp-poc-limits
```

> **No imperative equivalent:** unlike shortcuts such as `configmap`, `secret` and `quota` under `oc create`/`kubectl create`, **there is no imperative subcommand for LimitRange**. It can only be created from a YAML/JSON manifest with `oc apply -f` (or `oc create -f`).

### Scenario 1: defaults are injected automatically

Create a throwaway pod without specifying resources:

```bash
oc run limit-test --image=busybox:1.36 --restart=Never -- sleep 3600
```

Look at the pod's resources — even though we set nothing, the LimitRange injected its `defaultRequest`/`default` values:

```bash
oc get pod limit-test -o jsonpath='{.spec.containers[0].resources}{"\n"}'
# ✅ Actual output: {"limits":{"cpu":"100m","memory":"128Mi"},"requests":{"cpu":"50m","memory":"64Mi"}}

oc delete pod limit-test
```

### Scenario 2: exceeding `max` (combined with a rollout)

Try to raise the nginx container's limit above the LimitRange's `max.cpu: 500m`:

```bash
oc set resources deployment/nginx -c nginx --limits=cpu=800m,memory=512Mi
oc get replicaset -l app=nginx
oc describe replicaset <new-nginx-replicaset-name>
```

Events show the admission rejection (✅ this exact message was seen in the live test; nginx stayed `2/2` ready):

```
Warning  FailedCreate  ...  Error creating: pods "nginx-xxxxxxxxxx-zzzzz" is forbidden:
maximum cpu usage per Container is 500m, but limit is 800m
```

This is the same mechanism we saw in **Deployment rollout/rollback**: the new ReplicaSet cannot create a healthy pod, so the Deployment keeps running on the old ReplicaSet and the service is not interrupted. To revert:

```bash
oc set resources deployment/nginx -c nginx \
  --requests=cpu=50m,memory=64Mi --limits=cpu=200m,memory=128Mi
```

(We could also go back to the old ReplicaSet with `oc rollout undo deployment/nginx` — but here the old ReplicaSet is still active, so nothing else is needed.)

### Going below `min` (briefly)

By the same logic, requesting less than `min.cpu: 25m` is rejected too:

```bash
oc set resources deployment/nginx -c nginx --requests=cpu=10m
# ✅ Actual output: pods "..." is forbidden: minimum cpu usage per Container is 25m, but request is 10m
```

### Deleting the LimitRange

```bash
oc delete limitrange ocp-poc-limits
# or
oc delete -f limitrange.yaml
```

In the GUI: **Administrator** perspective → **Administration > LimitRanges** → `⋮` menu on the row → **Delete LimitRange**.

> Note: deleting a LimitRange does **not** change the resources of pods that already exist (they were injected at creation); default/min/max checks simply stop applying to new pods.

---

## Before Moving to the Next Guide

The `ocp-poc` namespace and the `nginx` app continue to be used in **[03 — Application Scenarios](../03-application-scenarios/README.md)**. However, **if the ResourceQuota and LimitRange from this guide are left in place**, the PostgreSQL pod in 03 cannot be created.

✅ **Actual output (moving to 03 without removing the quota):** `Create Pod postgres-0 in StatefulSet postgres failed error: pods "postgres-0" is forbidden: exceeded quota: ocp-poc-quota, requested: limits.cpu=300m, used: limits.cpu=900m, limited: limits.cpu=1`

So before moving to 03, remove the HPA demo, the quota and the limitrange, and revoke the `anyuid` grant:

```bash
oc delete -f hpa.yaml -f resourcequota.yaml -f limitrange.yaml
oc adm policy remove-scc-from-user anyuid -z default -n ocp-poc
```

After this step, the StatefulSet that was waiting because of the quota recovers on its own: in the live test `postgres-0` was `Running` within 52 s.

## Cleanup (at the end of the POC)

```bash
oc delete project ocp-poc
```
