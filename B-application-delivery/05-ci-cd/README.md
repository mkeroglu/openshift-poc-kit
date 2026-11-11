# 05 — CI/CD (Pipelines + GitOps)

> 🇹🇷 [Türkçe](README.tr.md) · [← 04 — Image Management](../../A-platform-foundation/04-image-management/README.md) · [POC flow](../../README.md) · [06 — Blue/Green Deployment →](../06-blue-green-deployment/README.md)

This section covers CI/CD and deployment strategies on OpenShift. Every step was **tested end to end on a live lab cluster** (OpenShift 4.22, OpenShift GitOps 1.22, OpenShift Pipelines 1.23); the GitOps test used this repository itself (GitHub, `main`) as its source:

- [x] GitOps deployment
- [x] Pipeline automation
- [x] Blue/Green deployment
- [x] Canary deployment

---

## GitOps Deployment

### What is GitOps?

GitOps is a delivery model in which the desired state of a cluster is kept in a **Git repository** as declarative YAML/Kustomize/Helm manifests, and an **operator/controller** (in this POC: **Argo CD**) continuously watches that repository and automatically makes the cluster's actual state match it. Core principles:

- **Single source of truth:** what runs in the cluster should be readable from the Git history. Instead of manual `oc apply` changes, changes are committed to Git.
- **Declarative:** the repository describes "what the end state should be", not "what to do".
- **Automatic sync + self-healing:** if a resource in the cluster is changed or deleted by hand, the GitOps controller brings it back to what is in Git (drift is fixed automatically).
- **Auditability:** every change is a Git commit — who changed what and when is clearly visible.

Tool: **Argo CD** (shipped on OpenShift as the **Red Hat OpenShift GitOps** operator).

### Installing the operator

**a) With YAML (reference — this is a cluster-wide resource, check whether it is already installed first):**

```bash
oc get csv -A | grep -i gitops   # already installed?
```

If not installed, `gitops-operator-subscription.yaml`:

```bash
oc apply -f gitops-operator-subscription.yaml
oc get csv -n openshift-operators | grep gitops
oc get ns openshift-gitops openshift-gitops-operator
```

Once the Subscription is created, the operator automatically creates the `openshift-gitops-operator` namespace and a default Argo CD instance (in the `openshift-gitops` namespace).

**b) With the Web Console (OperatorHub):**

1. In the **Administrator** perspective go to **Operators > OperatorHub**.
2. Search for **"Red Hat OpenShift GitOps"** and select it.
3. Click **Install**.
4. Continue to **Install** with the defaults (**Update Channel: latest**, **Installation Mode: All namespaces**, **Automatic** approval).
5. When the installation finishes, an Argo CD shortcut appears in the application launcher (the grid menu in the top navigation bar); it links to the Argo CD instance's Route.

### Accessing the Argo CD UI

```bash
oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='{.spec.host}{"\n"}'
```

The default username is `admin`; the password is stored in an automatically created secret:

```bash
oc extract secret/openshift-gitops-cluster -n openshift-gitops --to=- --keys=admin.password
```

CLI login:

```bash
argocd login <argocd-route-host> --username admin --password <password-from-above> --insecure
```

### Adding a repository to Argo CD (from the UI)

1. In the Argo CD UI go to **Settings > Repositories** in the left menu.
2. Click **+ CONNECT REPO**.
3. Choose the connection method:
   - **VIA HTTPS**: Repository URL (`https://.../repo.git`), plus Username + Password/Token if needed.
   - **VIA SSH**: Repository URL (`git@host:org/repo.git`) and an SSH private key.
4. Save with **CONNECT** — if the connection works, the repository appears in the list with a green connection status.

> For a private repository: over HTTPS a **personal access token** (instead of a password) is usually recommended; over SSH, a **deploy key** with read-only access to the repo.

### Creating an Application — with YAML

`argocd-application.yaml` (targets the `app/overlays/dev` directory of this repo):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gitops-demo-dev
  namespace: openshift-gitops
spec:
  project: default
  source:
    repoURL: https://github.com/mkeroglu/openshift-poc-kit.git
    targetRevision: main
    path: B-application-delivery/05-ci-cd/app/overlays/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: ocp-poc-gitops-dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

```bash
# works as-is against this repo; point repoURL at your fork to try your own changes:
oc apply -f argocd-application.yaml
oc get applications.argoproj.io -n openshift-gitops
```

✅ **Actual output:** **5 s** after the Application was created: `Sync: Synced, Health: Healthy`, revision = the latest commit in the repo. `CreateNamespace=true` created the `ocp-poc-gitops-dev` namespace; the Deployment (`1/1`), Service and edge Route arrived (`https://...` → `200`). A second Application from the same repo targeting `overlays/prod` came up in `ocp-poc-gitops-prod` with **3/3** replicas.

> **If you get a permission error:** on some installations the default `openshift-gitops` Argo CD instance can only manage its own namespace and namespaces labeled `argocd.argoproj.io/managed-by=openshift-gitops`. If sync fails with `... is forbidden`, label the target namespace: `oc label namespace <ns> argocd.argoproj.io/managed-by=openshift-gitops`. (For a namespace created with `CreateNamespace=true`, the label can be set with `managedNamespaceMetadata`.)

> **Note:** an `Application` resource may exist with the same name (`applications`) in both the `argoproj.io` API group and the `app.k8s.io` group from another operator. If it is ambiguous, use the full resource name: `oc get applications.argoproj.io`.

### Creating an Application — from the UI

1. On the Argo CD UI home page click **+ NEW APP**.
2. **GENERAL**: Application Name, Project (`default`), Sync Policy (`Automatic` or `Manual`).
3. **SOURCE**: pick the Repository URL you just added, Revision (e.g. `main`), Path (e.g. `B-application-delivery/05-ci-cd/app/overlays/dev`).
4. **DESTINATION**: Cluster URL (`https://kubernetes.default.svc` — the same cluster), Namespace (e.g. `ocp-poc-gitops-dev`).
5. The Kustomize path is detected automatically (no extra field needed).
6. Create the Application with **CREATE**; then press **SYNC** to trigger the first sync (automatic if you chose the Automatic policy).

### How the repository should be laid out

A repository managed with GitOps is typically laid out as **Kustomize base + overlays** — the `app/` directory in this repo is a live example:

```
app/
├── base/
│   ├── kustomization.yaml   # shared resources
│   ├── deployment.yaml
│   ├── service.yaml
│   └── route.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml   # references base, changes namespace + replica count per environment
    └── prod/
        └── kustomization.yaml
```

- **`base/`**: shared manifests, independent of the environment.
- **`overlays/<env>/`**: patches only the differences for that environment (namespace, replica count, image tag, resource limits, etc.) through `kustomization.yaml`; it does not change `base/`.
- Each Argo CD `Application` targets **one** of these overlay directories with `spec.source.path` (e.g. `app/overlays/dev` for dev, `app/overlays/prod` for prod) — so the different environments of the same app are managed from the same repo with different `Application` resources.

### Sync

**Manual vs automatic:**

- If `syncPolicy` is not set, the Application stays **manual** — changes in Git are **not applied automatically**; you need to press **SYNC** in the UI (or use `argocd app sync <name>` / `oc patch`).
- If `syncPolicy.automated` is set (as in `argocd-application.yaml` above), Argo CD applies every change in Git to the cluster **automatically**.

**Important sub-options:**

| Field | What it does |
|---|---|
| `automated.selfHeal: true` | If a resource in the cluster drifts from Git **by hand** (e.g. someone runs `oc edit`/`oc delete`), Argo CD automatically restores it to the Git version. |
| `automated.prune: true` | If a resource is **deleted** from Git, its counterpart in the cluster is deleted automatically. |
| `syncOptions: [CreateNamespace=true]` | Creates the target namespace if it does not exist. |

**`selfHeal` behavior verified live** — ✅ **Actual output:**

| Manual change | Argo CD's reaction |
|---|---|
| `oc delete svc gitops-demo` | Service recreated from Git within **2 s** |
| `oc scale deploy/gitops-demo --replicas=4` (1 in Git) | Replicas brought back to 1 within **8 s** |

In both cases the Application stayed `Synced/Healthy`; no manual intervention was needed.

**Triggering a manual sync:**

```bash
argocd app sync gitops-demo-dev
# or from the UI: click the Application card and press SYNC
```

**Checking the status:**

```bash
oc get applications.argoproj.io gitops-demo-dev -n openshift-gitops -o jsonpath='Sync: {.status.sync.status}, Health: {.status.health.status}{"\n"}'
```

`Sync` takes values like `Synced`/`OutOfSync`, and `Health` values like `Healthy`/`Progressing`/`Degraded`.

### Cleanup

```bash
oc delete applications.argoproj.io gitops-demo-dev -n openshift-gitops
oc delete namespace ocp-poc-gitops-dev ocp-poc-gitops-prod --ignore-not-found
```

## Pipeline Automation

### What are Tekton / OpenShift Pipelines?

**OpenShift Pipelines** is the OpenShift-integrated distribution of **Tekton**, a Kubernetes-native CI/CD engine. Concepts:

- **Task**: a single unit of work (e.g. "pull code from git", "build an image"). Contains one or more **Steps** (each a separate container).
- **Pipeline**: a workflow definition that links several Tasks sequentially (`runAfter`) or in parallel.
- **PipelineRun**: **a single execution** of a Pipeline with concrete parameters (each PipelineRun creates its own Pods).
- **TaskRun**: a concrete execution of a Task (created automatically inside a PipelineRun).
- **Workspace**: disk space shared between Tasks — e.g. the source code downloaded by the `git-clone` Task is written to a shared workspace so the `buildah` Task can read it.
- **Trigger** (`EventListener` + `TriggerBinding` + `TriggerTemplate`): the mechanism that automatically creates a new PipelineRun when a Git webhook arrives.

### Installing the operator

**a) With YAML (reference — cluster-wide, check whether it is installed first):**

```bash
oc get csv -A | grep -i pipelines
```

If not installed:

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-pipelines-operator
  namespace: openshift-operators
spec:
  channel: latest
  name: openshift-pipelines-operator-rh
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  installPlanApproval: Automatic
```

```bash
oc apply -f <file-above>
oc get ns openshift-pipelines
```

**b) With the Web Console (OperatorHub):**

1. In the **Administrator** perspective go to **Operators > OperatorHub**.
2. Search for and select the **"Red Hat OpenShift Pipelines"** operator.
3. **Install** → continue with the defaults (**All namespaces**, **Automatic** approval).
4. When installation finishes, the `openshift-pipelines` namespace is created with a ready-made **Task** library inside.

### Built-in Tasks

There used to be a separate resource type called `ClusterTask`; it is **deprecated** in current versions. Instead, common Tasks (`git-clone`, `buildah`, `openshift-client`, `s2i-*`, `maven`, `kn`, etc.) come installed as regular `Task` resources directly **in the `openshift-pipelines` namespace** and are referenced from a Pipeline with **`resolver: cluster`**:

```bash
oc get tasks -n openshift-pipelines
```

```yaml
taskRef:
  resolver: cluster
  params:
    - name: kind
      value: task
    - name: name
      value: git-clone
    - name: namespace
      value: openshift-pipelines
```

### Example Pipeline: build-and-deploy

`pipeline-build-deploy.yaml` — a realistic three-step "clone → build & push → deploy" flow:

1. **fetch-source** (`git-clone` Task) — clones the Git repository given as a parameter into `shared-workspace`.
2. **build-and-push** (`buildah` Task, `runAfter: fetch-source`) — builds the `Dockerfile` in the cloned source and pushes it to the address in the `IMAGE` parameter (e.g. the internal registry).
3. **deploy** (`openshift-client` Task, `runAfter: build-and-push`) — rolls out the new image by restarting the target Deployment with `oc rollout restart`.

Pipeline parameters: `git-url`, `git-revision`, `context-dir` (subdirectory containing the Dockerfile, `.` for the root), `image`, `deployment-name`.

```bash
oc apply -f pipeline-build-deploy.yaml
```

### Creating the target application first

Because the Pipeline's `deploy` step runs `oc rollout restart`, the target **Deployment must exist before the pipeline runs for the first time** (rollout restart does not create a missing Deployment, it only restarts an existing one). `pipeline-target-app.yaml` provides it — set your namespace and apply:

```bash
oc apply -f pipeline-target-app.yaml
# it is normal for the pod to be in ImagePullBackOff at first — the image has never been pushed yet
```

> The container uses `imagePullPolicy: Always` because every build uses the same `:latest` tag — without it, the node may keep using the old image it already cached.

### Before the first run in a new namespace: check `pipelines-scc`

If you trigger a PipelineRun **in a brand-new namespace** right after creating the Pipeline, the `fetch-source` step may fail with `Permission denied` (`/workspace/output/.git`), or `build-and-push` may fail writing `.docker`/`.config`. Cause: the `git-clone` and `buildah` Tasks are designed to run with the `pipelines-scc` SCC, but the OpenShift Pipelines operator provisions the `pipeline` ServiceAccount and `pipelines-scc-rolebinding` for the namespace **a few seconds after the Pipeline resource is created** — if a PipelineRun starts before that, the pods fall back to a more restrictive SCC (`restricted-v2`) and hit these errors.

After `oc apply -f pipeline-build-deploy.yaml`, verify the binding exists before triggering the PipelineRun:

```bash
oc get rolebinding pipelines-scc-rolebinding -n <namespace>
```

If it returns `NotFound`, wait a few seconds and try again (✅ in the live test the binding appeared **8 s** after the Pipeline was created). If the binding does not appear or the problem persists, you can also fix it by binding `pipelines-scc` to the `pipeline` ServiceAccount by hand:

```bash
oc adm policy add-scc-to-user pipelines-scc -z pipeline -n <namespace>
```

### Running with a PipelineRun (CLI)

`pipelinerun.yaml` — after replacing the namespace in the `image` field with your own:

```bash
oc create -f pipelinerun.yaml
oc get pipelinerun -w
```

Following the logs:

```bash
# if the tkn CLI is installed (download the `tkn` binary as the OpenShift Pipelines CLI):
tkn pipelinerun logs -f --last

# without tkn, directly with oc:
oc get pods -l tekton.dev/pipelineRun=<pipelinerun-name>
oc logs -f <pod-name> --all-containers
```

✅ **Actual output:** `Succeeded: Tasks Completed: 3 (Failed: 0, Cancelled 0)`, **164 s** in total:

| Task | Duration |
|---|---|
| `fetch-source` (git-clone) | 39 s |
| `build-and-push` (buildah → internal registry) | 1 min 50 s |
| `deploy` (`oc rollout restart` + status) | 8 s |

The `pipelines-demo` ImageStream was created; the Deployment that had been waiting in `ImagePullBackOff` became `Running` with the new image. Check: `curl http://<route>/vote` → `{"a":0,"b":0}` (`200`). (The root path `/` of this sample app returns `404`; this is expected.)

### Running from the UI (Web Console)

1. In the **Developer** perspective go to **Pipelines** in the left menu.
2. Click the `build-and-deploy` Pipeline and choose **Start** at the top right.
3. In the form, enter the parameters (`git-url`, `git-revision`, `context-dir`, `image`, `deployment-name`) and a **VolumeClaimTemplate** for the Workspace (size, e.g. 1Gi).
4. Trigger the PipelineRun with **Start** — progress is shown visually with a separate column/step for each Task (DAG view); click a step to see its live log stream.

### Automatic triggering with Triggers (webhook)

Instead of starting the pipeline by hand on every commit, it is triggered automatically by the Git provider's (GitHub/GitLab/Bitbucket/Gitea) **webhook**:

- **TriggerBinding**: maps fields in the webhook payload (`body.repository.clone_url`, `body.after`) to parameters.
- **TriggerTemplate**: the template of the PipelineRun to create with those parameters.
- **EventListener**: listens for the webhook through a Service/Route. An **Interceptor** filters the event type (e.g. only `push`) and can verify the webhook signature.

Flow: `Git push → webhook → EventListener Route → interceptor (filter/signature) → TriggerBinding → TriggerTemplate → new PipelineRun`.

`triggers.yaml` contains all of these (replace `REPLACE_ME_NAMESPACE` in the TriggerTemplate with your namespace):

```bash
NS=<namespace>
# For the EventListener to reach cluster-scoped interceptors (create BEFORE the EventListener):
oc create clusterrolebinding $NS-trigger-sa-clusterroles \
  --clusterrole=tekton-triggers-eventlistener-clusterroles --serviceaccount=$NS:pipeline-trigger-sa
sed "s/REPLACE_ME_NAMESPACE/$NS/" triggers.yaml | oc apply -n $NS -f -
oc get eventlistener build-and-deploy-listener -n $NS        # READY=True
oc expose svc el-build-and-deploy-listener -n $NS            # webhook address
oc get route el-build-and-deploy-listener -n $NS -o jsonpath='{.spec.host}'
```

Webhook on the Git provider: **Payload URL** = `http://<route above>`, **Content type** = `application/json`, **Events** = `push`. To verify the webhook signature, create a secret and add it to the interceptor:

```yaml
interceptors:
  - ref: {name: github}
    params:
      - name: secretRef
        value: {secretName: github-webhook-secret, secretKey: token}
      - name: eventTypes
        value: ["push"]
```

> If the ClusterRoleBinding is created after the EventListener, the EventListener pod gets a permission error on first start and restarts, and `READY` shows `False (MinimumReplicasUnavailable)` for a few seconds. Creating them in order avoids this.

**Test (without a Git provider, simulating the webhook):**

```bash
EL=$(oc get route el-build-and-deploy-listener -n $NS -o jsonpath='{.spec.host}')
# non-push event -> should be filtered out
curl -s -X POST http://$EL -H 'Content-Type: application/json' -H 'X-GitHub-Event: issues' -d '{}'
# push event -> should create a PipelineRun
curl -s -X POST http://$EL -H 'Content-Type: application/json' -H 'X-GitHub-Event: push' \
  -d '{"after":"master","repository":{"clone_url":"https://github.com/openshift/pipelines-vote-api.git"}}'
oc get pipelinerun -n $NS
```

✅ **Actual output:** both requests returned `{"eventListener":"build-and-deploy-listener",...,"eventID":"..."}` (the EventListener accepts the request; filtering happens behind it). A new PipelineRun (`build-and-deploy-webhook-9nxxz`) was created only for the `push` event; its parameters came from the payload (`https://github.com/openshift/pipelines-vote-api.git @ master`) and it `Succeeded` in **55 s**. The `issues` event did not create a PipelineRun.

### Cleanup

```bash
oc delete pipelinerun -l tekton.dev/pipeline=build-and-deploy
oc delete -f triggers.yaml
oc delete clusterrolebinding <namespace>-trigger-sa-clusterroles
oc delete -f pipeline-target-app.yaml
oc delete -f pipeline-build-deploy.yaml
```

---

## Blue/Green Deployment

Moved to a separate document — see **[06 — Blue/Green Deployment](../06-blue-green-deployment/README.md)** (concept, YAMLs, live test output and the router delay pitfall).

---

## Canary Deployment

Moved to a separate document — see **[07 — Canary Deployment](../07-canary-deployment/README.md)** (concept, Route `alternateBackends`/`weight` configuration, and a live test of the gradual `90/10 → 50/50 → 100/0` shift with actually measured request distributions).
