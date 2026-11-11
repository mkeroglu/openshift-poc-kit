# 11 — Security Tests (SCC/PSA + Image Signing)

> 🇹🇷 [Türkçe](README.tr.md) · [← 10 — Multus CNI](../10-multus-cni/README.md) · [POC flow](../../README.md) · [12 — Logging + LokiStack →](../../D-operations/12-logging/README.md)

This document covers two separate security topics:

1. **[SCC / PSA Policies](#1-scc--psa-policies)** — **tested live** on a lab cluster (OpenShift 4.22) in the `poc-psa-scc-test` namespace.
2. **[Image Signing](#2-image-signing)** — signing with cosign + a namespace-scoped `ImagePolicy`; **tested live** in the `poc-image-signing` namespace (no reboot, only a CRI-O reload).

---

## 1. SCC / PSA Policies

### Concept: Two Independent Gates

On OpenShift, for a pod to do something privileged (run as root, privileged mode, hostPath, etc.) it must pass **two separate, independent** admission gates:

1. **PSA (Pod Security Admission)** — the upstream Kubernetes standard. It statically checks the pod **spec** against the namespace's `pod-security.kubernetes.io/enforce` label (`restricted` / `baseline` / `privileged`). It looks only at the pod definition, **no matter who runs it**.
2. **SCC (SecurityContextConstraints)** — OpenShift-specific. It looks at **who creates the pod and/or which ServiceAccount it runs as**, and searches the SCCs that identity is **allowed to use** for one that fits the pod spec.

**The pod does not run unless both say "yes".** The tests below prove this with three scenarios.

> **⚠️ Critical methodology note (discovered in the live test):** SCC admission checks the permissions of the **user/service account making the request** — not only the pod's `spec.serviceAccountName`, but also **who runs `oc apply`/`oc create`**. While testing this POC as `system:admin` (cluster-admin), the tests wrongly produced "allowed" results because cluster-admin may use **every SCC** — they did not reflect the restriction a real (non-admin) user/SA of the namespace would see. **For a correct test, impersonation with `oc create --as=system:serviceaccount:<ns>:<sa>` is required** — otherwise your own admin permissions silently bypass the restriction you are testing.

### Test Environment

```bash
N=poc-psa-scc-test
oc create ns $N
oc label namespace $N pod-security.kubernetes.io/enforce=restricted --overwrite

# A realistic identity that is NOT cluster-admin: an SA with only "edit"
oc create sa tester-sa -n $N
oc create rolebinding tester-edit -n $N --clusterrole=edit --serviceaccount="${N}:tester-sa"
```

> **zsh users:** don't write `$N:tester-sa` — zsh treats `:t` as a variable modifier and mangles the name (`...-testester-sa`). Use the `"${N}:tester-sa"` form.

The pod under test (the same in every test; only the namespace label and the SCC grant change):

```yaml
# root-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: root-test
  namespace: poc-psa-scc-test
spec:
  containers:
    - name: test
      image: busybox:1.36
      command: ["id"]
      securityContext:
        runAsUser: 0
```

### Test 1: No extra SCC grant, PSA = `restricted` → should be rejected at the SCC layer

```bash
oc create -f root-pod.yaml --as=system:serviceaccount:poc-psa-scc-test:tester-sa
```

✅ **Actual output (shortened):**

```
Error from server (Forbidden): ... pods "root-test" is forbidden: unable to validate against any security context constraint:
[... "anyuid": Forbidden: not usable by user or serviceaccount ...
 provider restricted-v2: .containers[0].runAsUser: Invalid value: 0: must be in the ranges: [1001570000, 1001579999]
 provider restricted-v3: .containers[0].runAsUser: Invalid value: 0: must be in the ranges: [1000, 65534]
 ... "privileged": Forbidden: not usable by user or serviceaccount]
```

The only suitable SCCs `tester-sa` can use (`restricted-v2`/`v3`) enforce the namespace's allocated UID range, and `0` is outside it, so it is rejected. All other SCCs (`anyuid`, `privileged`, etc.) are already "not usable by user or serviceaccount" — this SA has no **permission** to use them at all.

### Test 2: SCC granted (`anyuid`), but PSA still `restricted` → should be rejected at the PSA layer

```bash
oc adm policy add-scc-to-user anyuid -z tester-sa -n poc-psa-scc-test

oc create -f root-pod.yaml --as=system:serviceaccount:poc-psa-scc-test:tester-sa
```

✅ **Actual output:**

```
Error from server (Forbidden): ... violates PodSecurity "restricted:latest":
allowPrivilegeEscalation != false, unrestricted capabilities, runAsNonRoot != true,
runAsUser=0 (container "test" must not set runAsUser=0), seccompProfile ...
```

**This proves the two layers really are independent:** SCC now allows it (`anyuid` was granted), but PSA still rejects it **on its own** — an SCC "yes" does not get past PSA.

### Test 3: Both allow it → it should really run as root

```bash
oc label namespace poc-psa-scc-test pod-security.kubernetes.io/enforce=baseline --overwrite

oc create -f root-pod.yaml --as=system:serviceaccount:poc-psa-scc-test:tester-sa
```

✅ **Actual output:**

```bash
oc logs root-test -n poc-psa-scc-test
# uid=0(root) gid=0(root) groups=0(root),10(wheel)

oc get pod root-test -n poc-psa-scc-test -o jsonpath='{.metadata.annotations.openshift\.io/scc}'
# anyuid
```

**It really ran as root** — once both PSA (`baseline`, allows root) and SCC (`anyuid`, granted explicitly) said "yes" at the same time.

### Summary

| # | PSA (`enforce`) | SCC grant (`tester-sa`) | Result |
|---|---|---|---|
| 1 | `restricted` | none (only `restricted-v2/v3`) | ❌ SCC rejected |
| 2 | `restricted` | `anyuid` granted | ❌ PSA rejected |
| 3 | `baseline` | `anyuid` granted | ✅ Ran, really `uid=0` |

✅ **Contrast test (why impersonation is required):** the same pod, with **no** `anyuid` on the SA (namespace `baseline`), **ran** when created with `oc create -f root-pod.yaml` as cluster-admin: `uid=0(root)`, `scc=anyuid`. The admin's own SCC permission silently bypassed the restriction; the SA's real permissions would not have given this result.

**Golden rule:** relaxing only one of these two layers is not enough; verify both separately in security tests — and **never test as cluster-admin**; impersonate the real runtime identity with `--as=system:serviceaccount:<ns>:<sa>`.

### Cleanup

```bash
oc delete namespace poc-psa-scc-test
```

---

## 2. Image Signing

### Concept

On OpenShift, the `ClusterImagePolicy` (cluster-wide) and `ImagePolicy` (namespace-scoped) resources (`config.openshift.io/v1`) **enforce sigstore signature verification**. If images pulled from a given registry/repository scope (`spec.scopes`) are not signed by the defined **root of trust** (public key, Fulcio+Rekor or your own PKI), **CRI-O does not pull the image** and the pod stays in `ImagePullBackOff`.

Behind the scenes the MCO turns the policy into a new rendered `MachineConfig` and rolls it out to all nodes (`/etc/containers/policy.json`, `/etc/crio/policies/<namespace>.json`, `/etc/containers/registries.d/`). For these files the **node disruption policy only does a `crio` reload**: no drain and no reboot.

This section was **tested end to end** on a lab cluster (OpenShift 4.22, 9 nodes) in the `poc-image-signing` namespace.

### Prerequisites

- `cosign` **v2.x** (tested: v2.5.3). ⚠️ cosign **v3** uses the new signature format (OCI referrers / bundle) by default. That format could not be written to the OpenShift internal registry: `PUT .../manifests/... UNKNOWN`. v2's classic `sha256-<digest>.sig` tag works fine.
- The image to sign must exist in the registry and be signed **by digest**.

### Step 1 — Key, namespace and two test images

```bash
N=poc-image-signing
COSIGN_PASSWORD="" cosign generate-key-pair          # cosign.key + cosign.pub
oc create namespace $N
oc create sa signer -n $N
oc policy add-role-to-user system:image-builder -z signer -n $N

REG=$(oc get route default-route -n openshift-image-registry -o jsonpath='{.spec.host}')
podman login --tls-verify=false -u signer -p "$(oc create token signer -n $N --duration=1h)" $REG
for app in signed-app unsigned-app; do
  podman tag <base-image> $REG/$N/$app:v1
  podman push --tls-verify=false --remove-signatures $REG/$N/$app:v1
done
```

> Without `--remove-signatures`, podman refuses to push with `Would invalidate signatures` because of the source image's own signatures.

Both images have **the same content (the same digest)**. The only difference is that the `signed-app` repository will be signed.

### Step 2 — Signing (inside the cluster, with the internal registry name)

The signature must be made with the reference the pod will pull (`image-registry.openshift-image-registry.svc:5000/...`). That is why cosign was run as a pod inside the cluster:

```bash
TOK=$(oc create token signer -n $N --duration=1h)
printf '{"auths":{"image-registry.openshift-image-registry.svc:5000":{"auth":"%s"}}}' \
  "$(printf 'signer:%s' "$TOK" | base64 -w0)" > config.json
oc create secret generic cosign-key -n $N --from-file=cosign.key --from-file=config.json; rm -f config.json

# Careful: 'oc get is ... .status.tags[0]' may also return the signature tag; take the digest from the istag
DIG=$(oc get istag signed-app:v1 -n $N -o jsonpath='{.image.metadata.name}')
IMG=image-registry.openshift-image-registry.svc:5000/$N/signed-app@$DIG

cat <<YAML | oc apply -f -
apiVersion: v1
kind: Pod
metadata: {name: cosign-sign, namespace: $N}
spec:
  restartPolicy: Never
  containers:
  - name: cosign
    image: ghcr.io/sigstore/cosign/cosign:v2.5.3
    args: ["sign","--yes","--key","/k/cosign.key","--tlog-upload=false","--allow-insecure-registry","$IMG"]
    env:
    - {name: COSIGN_PASSWORD, value: ""}
    - {name: DOCKER_CONFIG, value: /k}
    volumeMounts: [{name: k, mountPath: /k}]
  volumes: [{name: k, secret: {secretName: cosign-key}}]
YAML
oc logs -f cosign-sign -n $N
```

✅ **Actual output:** the signature was written to the registry as the `signed-app:sha256-<digest>.sig` tag. Signature payload:

```json
"critical": {
  "identity": { "docker-reference": "image-registry.openshift-image-registry.svc:5000/poc-image-signing/signed-app" },
  "image":    { "docker-manifest-digest": "sha256:7b4cbb00..." },
  "type": "cosign container image signature"
}
```

The `docker-reference` field has **no tag**, only the repository. This determines the `matchPolicy` choice in Step 3.

### Step 3 — ImagePolicy

`image-policy.yaml` (keyData = `base64 -w0 cosign.pub`):

```bash
sed "s|REPLACE_ME_COSIGN_PUB_BASE64|$(base64 -w0 cosign.pub)|" image-policy.yaml | oc apply -f -
oc get mcp -w        # watch the rollout
```

✅ **Actual output (rollout):**

- ~95 s after applying the policy, all 3 MCPs (`master`, `worker` and a custom worker pool) were `UPDATED=True`.
- No node went `NotReady`, and the **boot IDs of all 9 nodes did not change** (no reboot).
- Changing (`patch`) and deleting the policy likewise completed in ~90 s without a reboot.
- The file created on the node is `/etc/crio/policies/poc-image-signing.json`, with content `{"type":"sigstoreSigned","keyData":"...","signedIdentity":{"type":"matchRepository"}}`.

> On older OpenShift versions, or on clusters where node disruption policies have been changed, the same change may trigger a **reboot**. Check before applying: `oc get machineconfiguration cluster -o jsonpath='{.status.nodeDisruptionPolicyStatus.clusterPolicies.files}'` → `Reload crio` should appear for the relevant files.

### Step 4 — Verification

```bash
R=image-registry.openshift-image-registry.svc:5000/$N
oc run signed-tag    -n $N --image=$R/signed-app:v1        --restart=Never --command -- sleep 3600
oc run signed-digest -n $N --image=$R/signed-app@$DIG      --restart=Never --command -- sleep 3600
oc run unsigned-tag  -n $N --image=$R/unsigned-app:v1      --restart=Never --command -- sleep 3600
oc get pods -n $N
oc get events -n $N --field-selector reason=Failed | grep -o 'rejected: [^;]*'
```

✅ **Actual output:**

| Pod | `MatchRepoDigestOrExact` | `MatchRepository` |
|---|---|---|
| `signed-app:v1` (tag) | ❌ `Signature for identity "...signed-app" is not accepted` | ✅ `Running` |
| `signed-app@sha256:...` (digest) | ❌ same error | ✅ `Running` |
| `unsigned-app:v1` (same digest, unsigned repo) | ❌ `A signature was required, but no signature exists` | ❌ same (expected) |

**Finding:** because `docker-reference` in cosign signatures contains only the repository, `MatchRepoDigestOrExact` (the default recommendation) **also rejected the signed image** in this test, both by tag and by digest. Use `matchPolicy: MatchRepository` for images signed with cosign. For images signed under a different registry name (e.g. the external route), `RemapIdentity` can be used.

The unsigned `unsigned-app` image was rejected even though it has **exactly the same content** as the signed image: verification looks at the signature in the relevant repository, not at the image content.

### Practical Recommendations

1. **Try it first with an `ImagePolicy` in a single namespace**, then move to a `ClusterImagePolicy` if needed. A cluster-wide policy affects every image in its scope: a scope that covers `openshift-*` images can break the cluster.
2. Add signing to the CI pipeline (see [05 — CI/CD](../../B-application-delivery/05-ci-cd/README.md)): build → push → `cosign sign <image>@<digest>`.
3. If you must keep the private key (`cosign.key`) in the cluster, store it only in the signing namespace as a restricted secret; in production prefer KMS or keyless (Fulcio/Rekor).

### Cleanup

```bash
oc delete imagepolicy require-signed-images -n poc-image-signing   # ~90 s MCP rollout, no reboot
oc get mcp                                                            # wait until all are UPDATED=True
oc delete namespace poc-image-signing
rm -f cosign.key cosign.pub; podman logout --all
```

✅ **Actual output:** the rollout after deletion took ~80 s; the `/etc/crio/policies/` directory on the node was emptied and boot IDs did not change.
