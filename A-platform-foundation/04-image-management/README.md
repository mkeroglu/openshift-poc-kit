# 04 — Image Management

> 🇹🇷 [Türkçe](README.tr.md) · [← 03 — Application Scenarios](../03-application-scenarios/README.md) · [POC flow](../../README.md) · [05 — CI/CD →](../../B-application-delivery/05-ci-cd/README.md)

This section covers container image management on OpenShift. Every step was **tested end to end on a live lab cluster** (OpenShift 4.22, RHACS 4.11):

- [x] Using the internal registry
- [x] External registry integration
- [x] Vulnerability scanning

---

## Using the Internal Registry

### What is the internal registry?

OpenShift ships with a container image registry integrated into every cluster (in the `openshift-image-registry` namespace, managed by the `image-registry` cluster operator). Key features:

- From inside the cluster it is reachable at the DNS name `image-registry.openshift-image-registry.svc:5000`; the pod's service account token is used automatically for authentication (a separate `imagePullSecret` is usually not needed).
- Each namespace has its own **ImageStreams**; when an image is pushed, the matching ImageStream is created/updated **automatically**.
- If `defaultRoute: true`, a Route is opened so images can also be pushed/pulled from outside the cluster (with `podman`/`docker`).

### Checking the current state

```bash
oc get co image-registry
oc get configs.imageregistry.operator.openshift.io cluster -o yaml
```

Fields to check in the output:

| Field | Value | Meaning |
|---|---|---|
| `managementState` | active if `Managed` | The operator manages the registry (if disabled you see `Removed`; see the enablement steps below) |
| `defaultRoute` | external access open if `true` | To get the route hostname: `oc get route default-route -n openshift-image-registry -o jsonpath='{.spec.host}'` |
| `storage` | `emptyDir` / `pvc` / `s3` etc. | **Warning:** `emptyDir` is **not persistent** — if the registry pod is rescheduled (node change, restart), all pushed images are lost. Acceptable for a POC/demo; production must use persistent storage (PVC, S3-compatible object storage, etc.). |

### Enabling the internal registry (if disabled)

In some installations (especially bare metal, where storage is not configured automatically) the registry operator leaves itself `Removed`. To enable it:

```bash
# 1) Define persistent storage (production recommendation — PVC based)
oc patch configs.imageregistry.operator.openshift.io cluster --type merge -p \
  '{"spec":{"storage":{"pvc":{"claim":""}}}}'
# if "claim" is left empty, the operator creates a PVC from the default StorageClass

# 2) Enable the registry
oc patch configs.imageregistry.operator.openshift.io cluster --type merge -p \
  '{"spec":{"managementState":"Managed"}}'

# 3) (optional) Open the route for external access
oc patch configs.imageregistry.operator.openshift.io cluster --type merge -p \
  '{"spec":{"defaultRoute":true}}'

# Verify
oc get co image-registry
oc get pods -n openshift-image-registry
```

The same settings can be applied at once with a manifest — see `imageregistry-config-example.yaml` for reference.

> **Important:** these commands require **cluster-admin** and change the **single** cluster-wide registry configuration (it is not namespace-scoped). If the registry is already `Managed`, apply these `managementState`/`storage` changes carefully on a shared cluster — they can disrupt image push/pull for other namespaces. The push/pull/ImageStream steps below are namespace-scoped and safe.

### Push/Pull usage (tested live)

**1) Log in to the registry** — with your own OpenShift OAuth token; no separate registry user/password is needed:

```bash
REG=$(oc get route default-route -n openshift-image-registry -o jsonpath='{.spec.host}')
podman login --tls-verify=false -u "$(oc whoami)" -p "$(oc whoami -t)" "$REG"
```

> **If you are connected with a certificate (kubeconfig/`system:admin`)**, `oc whoami -t` returns empty (`error: no token is currently in use for this session`) because there is no OAuth token. In that case — and always for CI/CD systems — use a ServiceAccount token:
>
> ```bash
> oc create sa registry-pusher -n ocp-poc
> oc policy add-role-to-user system:image-builder -z registry-pusher -n ocp-poc   # push + pull
> podman login --tls-verify=false -u registry-pusher -p "$(oc create token registry-pusher -n ocp-poc --duration=1h)" "$REG"
> ```
>
> ✅ **Actual output:** `Login Succeeded!`

> `--tls-verify=false`: this route's certificate is not signed by a well-known CA (cluster-internal/self-signed CA). In production, add the cluster CA to the client's trust store and drop this flag.

**2) Push an image under your namespace:**

```bash
podman pull docker.io/library/alpine:3.20
podman tag docker.io/library/alpine:3.20 "$REG/ocp-poc/demo-alpine:latest"
podman push --tls-verify=false "$REG/ocp-poc/demo-alpine:latest"
```

> To push, having at least the `edit` role in that namespace is enough.

**3) Verify that the ImageStream was created automatically:**

```bash
oc get imagestream -n ocp-poc
oc describe imagestream demo-alpine -n ocp-poc
```

The push automatically creates/updates an **ImageStream** (`demo-alpine`) in the namespace — you do not need to run `oc create imagestream`.

**4) Use the image from inside the cluster, with no need for the external route:**

```bash
oc run demo-alpine-test --image=image-registry.openshift-image-registry.svc:5000/ocp-poc/demo-alpine:latest --restart=Never -- sleep 3600
oc get pod demo-alpine-test
```

Pods inside the cluster reach the internal registry through the **service DNS**; authentication happens automatically with the pod's service account token — within the same namespace no separate `imagePullSecret` is needed.

**5) Using short image names (image lookup policy):**

By default you cannot write a short name such as `image: demo-alpine:latest` in a Pod/Deployment — the full internal registry pull spec is needed. To make this easier, enable the ImageStream's **local lookup** policy:

```bash
oc set image-lookup demo-alpine
oc run demo-alpine-short --image=demo-alpine:latest --restart=Never -- sleep 3600
oc get pod demo-alpine-short -o jsonpath='{.spec.containers[0].image}'
# output: image-registry.openshift-image-registry.svc:5000/ocp-poc/demo-alpine@sha256:...
```

OpenShift resolves the short name to the full internal registry pull spec (including the digest) through an admission webhook.

**6) Pull test from outside the cluster:**

```bash
podman pull --tls-verify=false "$REG/ocp-poc/demo-alpine:latest"
```

✅ **Actual output:** after the push, the `demo-alpine` ImageStream was created automatically with the `latest` tag. The `demo-alpine-test` pod pulled it over the service DNS and became `Ready`. After `oc set image-lookup`, the image of the pod created with the short name (`demo-alpine:latest`) resolved to `image-registry.openshift-image-registry.svc:5000/ocp-poc/demo-alpine@sha256:26ba97...`. The local image was deleted and pulled again through the route.

### Cleanup

```bash
oc delete imagestream demo-alpine -n ocp-poc
podman rmi -f docker.io/library/alpine:3.20 "$REG/ocp-poc/demo-alpine:latest"
podman logout "$REG"
```

## External Registry Integration

Whatever registry your environment uses (Quay, Harbor, Artifactory, Nexus, Docker Hub, GHCR, etc.), the mechanism is the same: public pull, private pull secret, ImageStream import. This section tests and documents that mechanism live; in your environment only the registry hostname and credentials change.

### Concept

- **Public images** need no extra authentication — they can be pulled directly from any external registry.
- **Private images** need a **Secret** of type `kubernetes.io/dockerconfigjson`, linked either to the pod's `imagePullSecrets` field or to the namespace's `default` service account.
- OpenShift can also track a tag in an external registry by "importing" it into an **ImageStream**; with `--scheduled=true`, periodic automatic updates (and related image-change triggers) are possible.

### 1) Pull from a public external registry (tested live)

A public image on Quay.io that needs no authentication:

```bash
oc run quay-demo --image=quay.io/prometheus/busybox:latest --restart=Never -- sleep 3600
oc get pod quay-demo
```

Result: `1/1 Running` — it ran directly without defining any secret.

### 2) Pull secret mechanism for a private registry (tested live)

First the mechanism is shown with **placeholder (invalid) credentials**: you can tell OpenShift reads the secret and **tries** to connect to the registry with credentials from the change in the error message. Then (part **e**) a successful private pull with **valid** credentials is shown end to end.

**a) Trying a private-style image without a secret:**

```bash
oc run priv-nosecret --image=docker.io/ocppoc/private-demo:latest --restart=Never -- sleep 10
```

Actual result (`oc describe pod`):

```
Failed to pull image "docker.io/ocppoc/private-demo:latest": ... requested access to the resource is denied
```

**b) Create a secret with `external-registry-pull-secret.yaml` and try again:**

```bash
oc apply -f external-registry-pull-secret.yaml
# or the imperative equivalent:
oc create secret docker-registry external-registry-cred \
  --docker-server=docker.io \
  --docker-username=<real-username> \
  --docker-password=<real-password-or-token> \
  --docker-email=<email>
```

Trying again with placeholder credentials gives a **different** error — proof that the secret is actually read and sent to the registry.

✅ **Actual output (Docker Hub, 2026):**

```
Failed to pull image "docker.io/ocppoc/private-demo:latest": ... initializing source docker://ocppoc/private-demo:latest:
Requesting bearer token: received unexpected HTTP status: 400 Bad Request
```

(`requested access to the resource is denied` → no credentials were sent at all; `Requesting bearer token ... 400` / `incorrect username or password` → credentials were sent but are **invalid**. The exact text may vary by registry and over time.)

**c) Two ways to link the secret:**

- **At the Pod/Deployment level** (only for that workload) — see `external-image-demo.yaml`:
  ```yaml
  spec:
    imagePullSecrets:
      - name: external-registry-cred
  ```
- **At the service account level** (automatic for all pods using the namespace's `default` SA, without listing it in the pod spec) — also tested this way, with the same result:
  ```bash
  oc secrets link default external-registry-cred --for=pull
  ```

**d) Full Deployment example:**

```bash
oc apply -f external-image-demo.yaml
oc get pods -l app=external-image-demo
# expected: ImagePullBackOff (the placeholder credentials are not real) —
# it becomes Running once real registry credentials are filled in.
```

**e) Private pull with valid credentials (end to end):**

To stand in for a real private registry, the cluster's own registry is pulled **from another namespace**. Pods in one namespace cannot pull images from another namespace without credentials, so it behaves like a private registry:

```bash
oc new-project ocp-poc-ext
IMG=image-registry.openshift-image-registry.svc:5000/ocp-poc/demo-alpine:latest

# 1) Without a secret
oc run nosecret --image=$IMG --restart=Never -n ocp-poc-ext -- sleep 3600

# 2) Token of a ServiceAccount with pull permission only = "registry username/password"
oc create sa image-reader -n ocp-poc
oc policy add-role-to-user system:image-puller -z image-reader -n ocp-poc
oc create secret docker-registry private-registry-cred -n ocp-poc-ext \
  --docker-server=image-registry.openshift-image-registry.svc:5000 \
  --docker-username=image-reader --docker-password="$(oc create token image-reader -n ocp-poc --duration=8760h)"

# 3a) At pod level (imagePullSecrets)  /  3b) By linking to the ServiceAccount
oc secrets link default private-registry-cred --for=pull -n ocp-poc-ext
oc run sa-linked --image=$IMG --restart=Never -n ocp-poc-ext -- sleep 3600
```

✅ **Actual output:**

| Case | Result |
|---|---|
| No secret | `Failed to pull image ...: authentication required` |
| Pod with `imagePullSecrets: [private-registry-cred]` | `Running` |
| Secret linked to the `default` SA, no secret in the pod spec | `Running` |

With your own registry, the only difference is that the `--docker-server/--docker-username/--docker-password` values belong to that registry (e.g. a Quay robot account).

### 3) Importing/tracking an external image with an ImageStream (tested live)

```bash
oc import-image external-busybox --from=quay.io/prometheus/busybox --confirm --scheduled=true
oc describe imagestream external-busybox
```

The actual output contains:

```
latest
  updates automatically from registry quay.io/prometheus/busybox
```

With `--scheduled=true`, OpenShift checks the external registry periodically (about every 15 minutes by default); if the source tag changes, the ImageStream is updated automatically — which can in turn fire a linked `ImageChangeTrigger` (e.g. a Build or DeploymentConfig).

### Cleanup

```bash
oc delete deployment external-image-demo
oc delete secret external-registry-cred
oc secrets unlink default external-registry-cred
oc delete imagestream external-busybox
oc delete pod quay-demo priv-nosecret priv-withsecret priv-sa-linked --ignore-not-found
oc delete project ocp-poc-ext
oc delete sa image-reader registry-pusher -n ocp-poc
```

## Vulnerability Scanning

Tool: **Red Hat Advanced Cluster Security (RHACS / StackRox)**. ACS has two main parts:

- **Central** — management UI, API, image/CVE database (runs in the `stackrox` namespace).
- **Scanner** — the component that indexes image layers (Indexer) and matches them against the known-CVE database (Matcher).

Optionally, a **SecuredCluster** (Sensor + Collector + Admission Controller) can be installed on each cluster for runtime monitoring and deploy-time policy enforcement (e.g. "images with critical CVEs cannot be deployed") — this is not required for image scanning alone.

### Checking the current state

```bash
oc get ns | grep -iE "stackrox|rhacs"
oc get central -n stackrox
oc get securedcluster -n stackrox
oc get route central -n stackrox
```

| Component | Expected state |
|---|---|
| **Central** | `AVAILABLE: True` — UI + API reachable |
| **Scanner** | Its pods `Running`, CVE database up to date |
| **SecuredCluster** (if any) | `AVAILABLE: True` — if not, check the error under `status.conditions` with `oc get securedcluster <name> -o yaml` |

If Central is not installed yet, install it with the **RHACS Operator** (from OperatorHub): first create a `Central` CR (Central services), then apply a `SecuredCluster` CR for each cluster to secure. Installation is out of scope for this POC — an existing or to-be-installed Central is used.

### Prerequisite for image scanning: Image Integration

> **If a SecuredCluster is installed**, ACS creates integrations automatically from the cluster's pull secrets and internal registry (such as `Autogenerated https://image-registry.openshift-image-registry.svc:5000 for cluster <name>`). In that case there is no need to add an integration for the internal registry by hand. The step below is for when there is no SecuredCluster or the automatic integration is not enough.

For Central to scan images in a registry, that registry must be defined as an **Image Integration** — otherwise the scan request is rejected with `no matching image registries found`. List the existing integrations:

```bash
CENTRAL=$(oc get route central -n stackrox -o jsonpath='{.spec.host}')
PASS="$(oc get secret central-htpasswd -n stackrox -o jsonpath='{.data.password}' | base64 -d)"
curl -sk -u "admin:$PASS" "https://$CENTRAL/v1/imageintegrations" | python3 -m json.tool
```

If the internal registry (or the external registry you will use) is not listed, fill in `acs-image-integration.json` with your own username/token and add it:

```bash
curl -sk -u "admin:$PASS" -X POST "https://$CENTRAL/v1/imageintegrations" \
  -H "Content-Type: application/json" \
  -d @acs-image-integration.json
```

> A personal `oc whoami -t` token can be used during a demo/setup, but **in production** use a **dedicated service account token** with only `system:image-puller` instead (personal tokens expire/can be revoked, which makes the integration fragile).

### Scanning an image with `roxctl`

If you don't have `roxctl`, download it from Central itself (the version will match Central exactly):

```bash
curl -sk -u "admin:$PASS" -o roxctl "https://$CENTRAL/api/cli/download/roxctl-linux" && chmod +x roxctl
export ROX_ADMIN_PASSWORD="$PASS"
```

```bash
roxctl image scan \
  -e "${CENTRAL}:443" --insecure-skip-tls-verify \
  -p "$PASS" \
  --image image-registry.openshift-image-registry.svc:5000/<namespace>/<image>:<tag> \
  --output table
```

> `--insecure-skip-tls-verify`: needed if the route certificate belongs to a self-signed/cluster-internal CA. For large/slow scans over the route, HAProxy's default idle timeout can cut the connection (`transport: EOF` error); in that case connect to Central directly with `oc port-forward svc/central 18443:443 -n stackrox` and use `-e localhost:18443` to get around the timeout.

Add `--force` to rescan instead of returning a cached old result.

✅ **Actual output:**

| Image | Time | Result |
|---|---|---|
| `registry.access.redhat.com/ubi8/ubi:8.4` (old, deliberately unpatched) | 22 s | `TOTAL-COMPONENTS: 112, TOTAL-VULNERABILITIES: 779, LOW: 285, MODERATE: 411, IMPORTANT: 83, CRITICAL: 0` |
| `quay.io/prometheus/busybox:latest` | <1 s | `TOTAL-COMPONENTS: 0` (a single static binary; no package database, expected) |
| `alpine:3.20` (Docker Hub and internal registry) | <5 s | `TOTAL-COMPONENTS: 0` — unexpected; see the checklist below |

In the lab, an index report was created for Alpine but the result came back empty. Both `Scanner V4` and `StackRox Scanner` integrations were defined in the environment, which matches item 3 below. Enterprise (UBI/RHEL-based) images were scanned without issues.

### If the result is empty: checklist

If a scan returns an empty result such as `TOTAL-COMPONENTS: 0`, check the following in order:

1. **Is the Image Integration correct?** — in the `curl .../v1/imageintegrations` output, the registry's `endpoint` field must exactly match the registry host of the scanned image (e.g. `docker.io` and `registry-1.docker.io` are different values; if they don't match, the integration is not used).
2. **Did the Indexer process the image?** — confirm the layers were read with `oc logs -n stackrox deploy/scanner-v4-indexer --since=5m | grep "manifest successfully scanned"`.
3. **Was the Matcher called?** — confirm the CVE matching step ran with `oc logs -n stackrox deploy/scanner-v4-matcher --since=5m | grep GetVulnerabilities`. If the Indexer succeeds but the Matcher is never called, there is a problem in Central's own enrichment pipeline (e.g. multiple scanner integrations conflicting) — contact the platform/operations team that runs ACS.
4. **Is the image registered in Central?** — check with `curl .../v1/images?query=Image:<image-name>` whether the image was processed into Central's own database.

### Cleanup

```bash
oc delete imagestream <test-image> -n <namespace> --ignore-not-found
podman rmi -f <local-image-references>
podman logout "$REG"
```
