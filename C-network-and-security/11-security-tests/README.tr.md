# 11 — Güvenlik Testleri (SCC/PSA + Image Signing)

> 🇬🇧 [English](README.md) · [← 10 — Multus CNI](../10-multus-cni/README.tr.md) · [POC akışı](../../README.tr.md) · [12 — Logging + LokiStack →](../../D-operations/12-logging/README.tr.md)

Bu doküman iki ayrı güvenlik konusunu kapsar:

1. **[SCC / PSA Politikaları](#1-scc--psa-politikaları)** — lab ortamında (OpenShift 4.22) `poc-psa-scc-test` namespace'inde **canlı test edilmiştir**.
2. **[Image Signing](#2-image-signing)** — cosign ile imzalama + namespace kapsamlı `ImagePolicy`; `poc-image-signing` namespace'inde **canlı test edilmiştir** (reboot yok, yalnızca CRI-O reload).

---

## 1. SCC / PSA Politikaları

### Kavram: İki Bağımsız Kapı

OpenShift'te bir pod'un ayrıcalıklı bir şey yapabilmesi (root çalışmak, privileged mod, hostPath vb.) için **iki ayrı, birbirinden bağımsız** admission kapısından geçmesi gerekir:

1. **PSA (Pod Security Admission)** — upstream Kubernetes standardı. Namespace'in `pod-security.kubernetes.io/enforce` label'ına göre (`restricted` / `baseline` / `privileged`) pod **spec**'ini statik olarak kontrol eder. **Kim çalıştırıyor olursa olsun**, sadece pod tanımına bakar.
2. **SCC (SecurityContextConstraints)** — OpenShift'e özgü. Pod'u **kimin oluşturduğuna ve/veya hangi ServiceAccount ile çalışacağına** bakarak, o kimliğin **kullanmaya yetkili olduğu** SCC'ler arasından pod spec'ine uyanı arar.

**İkisi de "evet" demeden pod çalışmaz.** Aşağıdaki testler bunu üç senaryoyla kanıtlıyor.

> **⚠️ Kritik metodoloji notu (canlı testte keşfedildi):** SCC admission, isteği yapan **kullanıcının/service account'ın** yetkisine bakar — pod'un `spec.serviceAccountName`'i değil, **`oc apply`/`oc create`'i kimin çalıştırdığı** da devreye girer. Bu POC'de `system:admin` (cluster-admin) ile test ederken, cluster-admin'in **her SCC'yi** kullanma yetkisi olduğu için testler yanlışlıkla "izin verildi" sonucu üretti — namespace'in gerçek (admin olmayan) bir kullanıcısının/SA'sının göreceği kısıtlamayı yansıtmıyordu. **Doğru test için `oc create --as=system:serviceaccount:<ns>:<sa>` ile impersonation şart** — aksi halde kendi admin yetkiniz test ettiğiniz kısıtlamayı sessizce by-pass eder.

### Test Ortamı

```bash
N=poc-psa-scc-test
oc create ns $N
oc label namespace $N pod-security.kubernetes.io/enforce=restricted --overwrite

# Cluster-admin OLMAYAN, gerçekçi bir kimlik: sadece "edit" yetkili bir SA
oc create sa tester-sa -n $N
oc create rolebinding tester-edit -n $N --clusterrole=edit --serviceaccount="${N}:tester-sa"
```

> **zsh kullanıcıları:** `$N:tester-sa` yazmayın — zsh `:t`'yi değişken düzenleyicisi sanar ve adı bozar (`...-testester-sa`). `"${N}:tester-sa"` biçimini kullanın.

Test edilen pod (hepsinde aynı, sadece namespace label'ı ve SCC izni değişiyor):

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

### Test 1: Hiçbir ek SCC izni yok, PSA = `restricted` → SCC katmanında reddedilmeli

```bash
oc create -f root-pod.yaml --as=system:serviceaccount:poc-psa-scc-test:tester-sa
```

✅ **Gerçek çıktı (kısaltılmış):**

```
Error from server (Forbidden): ... pods "root-test" is forbidden: unable to validate against any security context constraint:
[... "anyuid": Forbidden: not usable by user or serviceaccount ...
 provider restricted-v2: .containers[0].runAsUser: Invalid value: 0: must be in the ranges: [1001570000, 1001579999]
 provider restricted-v3: .containers[0].runAsUser: Invalid value: 0: must be in the ranges: [1000, 65534]
 ... "privileged": Forbidden: not usable by user or serviceaccount]
```

`tester-sa`'nın erişebildiği tek uygun SCC'ler (`restricted-v2`/`v3`) namespace'in ayrılmış UID aralığını zorluyor, `0` bu aralıkta olmadığı için reddediliyor. Diğer tüm SCC'ler (`anyuid`, `privileged` vb.) zaten "not usable by user or serviceaccount" — yani bu SA'nın onları kullanma **yetkisi** hiç yok.

### Test 2: SCC izni verildi (`anyuid`), ama PSA hâlâ `restricted` → PSA katmanında reddedilmeli

```bash
oc adm policy add-scc-to-user anyuid -z tester-sa -n poc-psa-scc-test

oc create -f root-pod.yaml --as=system:serviceaccount:poc-psa-scc-test:tester-sa
```

✅ **Gerçek çıktı:**

```
Error from server (Forbidden): ... violates PodSecurity "restricted:latest":
allowPrivilegeEscalation != false, unrestricted capabilities, runAsNonRoot != true,
runAsUser=0 (container "test" must not set runAsUser=0), seccompProfile ...
```

**Bu, iki katmanın gerçekten bağımsız olduğunun kanıtı:** SCC artık izin veriyor (`anyuid` atandı), ama PSA hâlâ **kendi başına** reddediyor — SCC'nin "evet" demesi PSA'yı atlatmıyor.

### Test 3: İkisi de izin veriyor → gerçekten root olarak çalışmalı

```bash
oc label namespace poc-psa-scc-test pod-security.kubernetes.io/enforce=baseline --overwrite

oc create -f root-pod.yaml --as=system:serviceaccount:poc-psa-scc-test:tester-sa
```

✅ **Gerçek çıktı:**

```bash
oc logs root-test -n poc-psa-scc-test
# uid=0(root) gid=0(root) groups=0(root),10(wheel)

oc get pod root-test -n poc-psa-scc-test -o jsonpath='{.metadata.annotations.openshift\.io/scc}'
# anyuid
```

**Gerçekten root olarak çalıştı** — hem PSA (`baseline`, root'a izin verir) hem SCC (`anyuid`, açıkça verildi) aynı anda "evet" dediğinde.

### Özet

| # | PSA (`enforce`) | SCC izni (`tester-sa`) | Sonuç |
|---|---|---|---|
| 1 | `restricted` | yok (sadece `restricted-v2/v3`) | ❌ SCC reddetti |
| 2 | `restricted` | `anyuid` verildi | ❌ PSA reddetti |
| 3 | `baseline` | `anyuid` verildi | ✅ Çalıştı, gerçekten `uid=0` |

✅ **Kontrast testi (neden impersonation şart):** Aynı pod, SA'da `anyuid` **yokken** (namespace `baseline`) cluster-admin kimliğiyle `oc create -f root-pod.yaml` ile oluşturulduğunda **çalıştı**: `uid=0(root)`, `scc=anyuid`. Admin'in kendi SCC yetkisi kısıtlamayı sessizce atladı; SA'nın gerçek yetkisi bu sonucu vermezdi.

**Altın kural:** Bu iki katmandan sadece birini gevşetmek yetmez, güvenlik testlerinde ikisini de ayrı ayrı doğrulayın — ve **asla cluster-admin kimliğiyle test etmeyin**, `--as=system:serviceaccount:<ns>:<sa>` ile gerçek çalışma zamanı kimliğini impersonate edin.

### Temizlik

```bash
oc delete namespace poc-psa-scc-test
```

---

## 2. Image Signing

### Kavram

OpenShift'te `ClusterImagePolicy` (cluster geneli) ve `ImagePolicy` (namespace kapsamlı) kaynakları (`config.openshift.io/v1`), **sigstore imza doğrulamasını zorunlu** kılar. Belirli bir registry/repository scope'undan (`spec.scopes`) çekilen imajlar, tanımlı **root of trust** ile (public key, Fulcio+Rekor ya da kendi PKI'nız) imzalı değilse **CRI-O imajı çekmez** ve pod `ImagePullBackOff`'ta kalır.

Arka planda MCO, politikayı yeni bir rendered `MachineConfig`'e çevirir ve tüm node'lara dağıtır (`/etc/containers/policy.json`, `/etc/crio/policies/<namespace>.json`, `/etc/containers/registries.d/`). Bu dosyalar için **node disruption policy yalnızca `crio` reload** yapar: drain ve reboot olmaz.

Bu bölüm lab ortamında (OpenShift 4.22, 9 node) `poc-image-signing` namespace'inde **uçtan uca test edilmiştir**.

### Ön Koşullar

- `cosign` **v2.x** (test: v2.5.3). ⚠️ cosign **v3** varsayılan olarak yeni imza biçimini (OCI referrers / bundle) kullanır. Bu biçim OpenShift internal registry'ye yazılamadı: `PUT .../manifests/... UNKNOWN`. v2'nin klasik `sha256-<digest>.sig` etiketi sorunsuz çalışır.
- İmzalanacak imajın registry'de bulunması ve **digest** ile imzalanması.

### Adım 1 — Anahtar, namespace ve iki test imajı

```bash
N=poc-image-signing
COSIGN_PASSWORD="" cosign generate-key-pair          # cosign.key + cosign.pub
oc create namespace $N
oc create sa signer -n $N
oc policy add-role-to-user system:image-builder -z signer -n $N

REG=$(oc get route default-route -n openshift-image-registry -o jsonpath='{.spec.host}')
podman login --tls-verify=false -u signer -p "$(oc create token signer -n $N --duration=1h)" $REG
for app in signed-app unsigned-app; do
  podman tag <taban-imaj> $REG/$N/$app:v1
  podman push --tls-verify=false --remove-signatures $REG/$N/$app:v1
done
```

> `--remove-signatures` olmadan podman, kaynak imajın kendi imzaları nedeniyle `Would invalidate signatures` hatası verip push etmez.

İki imaj **aynı içeriğe (aynı digest'e)** sahiptir. Fark, yalnızca `signed-app` repository'sinin imzalanacak olmasıdır.

### Adım 2 — İmzalama (cluster içinde, internal registry adıyla)

İmza, pod'un çekeceği referansla (`image-registry.openshift-image-registry.svc:5000/...`) atılmalıdır. Bu yüzden cosign cluster içinde bir pod olarak çalıştırıldı:

```bash
TOK=$(oc create token signer -n $N --duration=1h)
printf '{"auths":{"image-registry.openshift-image-registry.svc:5000":{"auth":"%s"}}}' \
  "$(printf 'signer:%s' "$TOK" | base64 -w0)" > config.json
oc create secret generic cosign-key -n $N --from-file=cosign.key --from-file=config.json; rm -f config.json

# Dikkat: 'oc get is ... .status.tags[0]' imza etiketini de döndürebilir; digest'i istag'den alın
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

✅ **Gerçek çıktı:** İmza `signed-app:sha256-<digest>.sig` etiketi olarak registry'ye yazıldı. İmza payload'ı:

```json
"critical": {
  "identity": { "docker-reference": "image-registry.openshift-image-registry.svc:5000/poc-image-signing/signed-app" },
  "image":    { "docker-manifest-digest": "sha256:7b4cbb00..." },
  "type": "cosign container image signature"
}
```

`docker-reference` alanında **tag yoktur**, yalnızca repository vardır. Bu, Adım 3'teki `matchPolicy` seçimini belirler.

### Adım 3 — ImagePolicy

`image-policy.yaml` (keyData = `base64 -w0 cosign.pub`):

```bash
sed "s|REPLACE_ME_COSIGN_PUB_BASE64|$(base64 -w0 cosign.pub)|" image-policy.yaml | oc apply -f -
oc get mcp -w        # rollout'u izleyin
```

✅ **Gerçek çıktı (rollout):**

- Politika uygulandıktan ~95 sn sonra 3 MCP de (`master`, `worker` ve özel bir worker havuzu) `UPDATED=True` oldu.
- Hiçbir node `NotReady` olmadı ve 9 node'un **boot ID'si değişmedi** (reboot yok).
- Politika değişikliği (`patch`) ve silme de aynı şekilde ~90 sn'de, reboot olmadan tamamlandı.
- Node'da oluşan dosya `/etc/crio/policies/poc-image-signing.json`, içeriği `{"type":"sigstoreSigned","keyData":"...","signedIdentity":{"type":"matchRepository"}}`.

> Eski OpenShift sürümlerinde ya da node disruption policy'leri değiştirilmiş cluster'larda aynı değişiklik **reboot** tetikleyebilir. Uygulamadan önce kontrol edin: `oc get machineconfiguration cluster -o jsonpath='{.status.nodeDisruptionPolicyStatus.clusterPolicies.files}'` → ilgili dosyalar için `Reload crio` görünmeli.

### Adım 4 — Doğrulama

```bash
R=image-registry.openshift-image-registry.svc:5000/$N
oc run signed-tag    -n $N --image=$R/signed-app:v1        --restart=Never --command -- sleep 3600
oc run signed-digest -n $N --image=$R/signed-app@$DIG      --restart=Never --command -- sleep 3600
oc run unsigned-tag  -n $N --image=$R/unsigned-app:v1      --restart=Never --command -- sleep 3600
oc get pods -n $N
oc get events -n $N --field-selector reason=Failed | grep -o 'rejected: [^;]*'
```

✅ **Gerçek çıktı:**

| Pod | `MatchRepoDigestOrExact` | `MatchRepository` |
|---|---|---|
| `signed-app:v1` (tag) | ❌ `Signature for identity "...signed-app" is not accepted` | ✅ `Running` |
| `signed-app@sha256:...` (digest) | ❌ aynı hata | ✅ `Running` |
| `unsigned-app:v1` (aynı digest, imzasız repo) | ❌ `A signature was required, but no signature exists` | ❌ aynı (beklenen) |

**Bulgu:** cosign imzalarında `docker-reference` yalnızca repository içerdiği için `MatchRepoDigestOrExact` (varsayılan öneri) bu testte hem tag hem digest ile **imzalı imajı da reddetti**. cosign ile imzalanan imajlarda `matchPolicy: MatchRepository` kullanın. Farklı bir registry adıyla (örn. dış route) imzalanan imajlar için `RemapIdentity` kullanılabilir.

İmzasız `unsigned-app` imajı, imzalı imajla **birebir aynı içeriğe** sahip olmasına rağmen reddedildi: doğrulama imaj içeriğine değil, ilgili repository'deki imzaya bakar.

### Pratik Öneriler

1. **Önce `ImagePolicy` ile tek bir namespace'te deneyin**, sonra gerekirse `ClusterImagePolicy`'ye geçin. Cluster geneli politika, scope'taki her imajı etkiler: `openshift-*` imajlarını kapsayan bir scope cluster'ı bozabilir.
2. İmzalamayı CI pipeline'ına ekleyin (bkz. [05 — CI/CD](../../B-application-delivery/05-ci-cd/README.tr.md)): build → push → `cosign sign <imaj>@<digest>`.
3. Özel anahtarı (`cosign.key`) cluster'da tutmanız gerekiyorsa yalnızca imzalama namespace'inde, kısıtlı bir secret olarak saklayın; üretimde KMS ya da keyless (Fulcio/Rekor) tercih edin.

### Temizlik

```bash
oc delete imagepolicy require-signed-images -n poc-image-signing   # ~90 sn MCP rollout, reboot yok
oc get mcp                                                            # hepsi UPDATED=True olana kadar bekleyin
oc delete namespace poc-image-signing
rm -f cosign.key cosign.pub; podman logout --all
```

✅ **Gerçek çıktı:** Silme sonrası rollout ~80 sn sürdü; node'daki `/etc/crio/policies/` dizini boşaldı ve boot ID'ler değişmedi.
