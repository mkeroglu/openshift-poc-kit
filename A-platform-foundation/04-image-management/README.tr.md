# 04 — Image Yönetimi

> 🇬🇧 [English](README.md) · [← 03 — Uygulama Senaryoları](../03-application-scenarios/README.tr.md) · [POC akışı](../../README.tr.md) · [05 — CI/CD →](../../B-application-delivery/05-ci-cd/README.tr.md)

Bu bölümde OpenShift üzerinde container image'larının yönetimi ele alınacaktır. Tüm adımlar lab ortamında (OpenShift 4.22, RHACS 4.11) **uçtan uca canlı test edilmiştir**:

- [x] Internal registry kullanımı
- [x] External registry entegrasyonu
- [x] Vulnerability scanning

---

## Internal Registry Kullanımı

### Internal Registry nedir?

OpenShift, her cluster'a entegre gelen bir container image registry ile birlikte kurulur (`openshift-image-registry` namespace'inde, `image-registry` cluster operator'ü tarafından yönetilir). Öne çıkan özellikleri:

- Cluster içinden `image-registry.openshift-image-registry.svc:5000` DNS adı üzerinden erişilir; kimlik doğrulama için pod'un service account token'ı otomatik kullanılır (ayrı bir `imagePullSecret` tanımlamaya genelde gerek yoktur).
- Her namespace kendi **ImageStream**'lerine sahiptir; bir image push edildiğinde ilgili ImageStream **otomatik** oluşturulur/güncellenir.
- `defaultRoute: true` ise cluster dışından da (`podman`/`docker` ile) push/pull yapılabilecek bir Route açılır.

### Mevcut durumu kontrol etme

```bash
oc get co image-registry
oc get configs.imageregistry.operator.openshift.io cluster -o yaml
```

Çıktıda kontrol edilmesi gereken alanlar:

| Alan | Değer | Anlamı |
|---|---|---|
| `managementState` | `Managed` ise aktif | Operator registry'i yönetiyor (kapalıysa `Removed` görürsünüz, bkz. aşağıdaki aktifleştirme adımları) |
| `defaultRoute` | `true` ise harici erişim açık | Route hostname'i öğrenmek için: `oc get route default-route -n openshift-image-registry -o jsonpath='{.spec.host}'` |
| `storage` | `emptyDir` / `pvc` / `s3` vb. | **Dikkat:** `emptyDir` ise **kalıcı değildir** — registry pod'u yeniden zamanlanırsa (node değişimi, restart) push edilen tüm imajlar kaybolur. POC/demo amaçlı kabul edilebilir; production'da mutlaka kalıcı storage (PVC, S3-uyumlu object storage vb.) kullanılmalıdır. |

### Internal Registry nasıl aktif edilir? (kapalıysa)

Bazı kurulumlarda (özellikle bare-metal, storage otomatik yapılandırılmadığı için) registry operator kendini `Removed` bırakır. Aktif etmek için:

```bash
# 1) Kalıcı storage tanımlayın (production önerisi — PVC tabanlı)
oc patch configs.imageregistry.operator.openshift.io cluster --type merge -p \
  '{"spec":{"storage":{"pvc":{"claim":""}}}}'
# "claim" boş bırakılırsa operator, varsayılan StorageClass'tan otomatik bir PVC oluşturur

# 2) Registry'i aktif edin
oc patch configs.imageregistry.operator.openshift.io cluster --type merge -p \
  '{"spec":{"managementState":"Managed"}}'

# 3) (opsiyonel) Harici erişim için route'u açın
oc patch configs.imageregistry.operator.openshift.io cluster --type merge -p \
  '{"spec":{"defaultRoute":true}}'

# Doğrulama
oc get co image-registry
oc get pods -n openshift-image-registry
```

Aynı ayarları tek seferde bir manifest ile de uygulayabilirsiniz — referans için `imageregistry-config-example.yaml` dosyasına bakın.

> **Önemli:** Bu komutlar **cluster-admin** yetkisi gerektirir ve cluster genelindeki **tek** registry yapılandırmasını değiştirir (namespace-scoped değildir). Registry zaten `Managed` durumdaysa bu `managementState`/`storage` değişikliklerini paylaşılan bir cluster'da dikkatli uygulayın — diğer namespace'lerin image push/pull işlemlerini kesintiye uğratma riski taşır. Aşağıdaki push/pull/ImageStream adımları namespace-scoped ve güvenlidir.

### Push/Pull ile kullanım (canlı test edildi)

**1) Registry'ye login olun** — kendi OpenShift OAuth token'ınızla, ayrı bir registry kullanıcı/parolasına gerek yoktur:

```bash
REG=$(oc get route default-route -n openshift-image-registry -o jsonpath='{.spec.host}')
podman login --tls-verify=false -u "$(oc whoami)" -p "$(oc whoami -t)" "$REG"
```

> **Sertifikayla (kubeconfig/`system:admin`) bağlıysanız** `oc whoami -t` boş döner (`error: no token is currently in use for this session`), çünkü OAuth token'ı yoktur. Bu durumda — ve CI/CD sistemleri için her zaman — bir ServiceAccount token'ı kullanın:
>
> ```bash
> oc create sa registry-pusher -n ocp-poc
> oc policy add-role-to-user system:image-builder -z registry-pusher -n ocp-poc   # push + pull
> podman login --tls-verify=false -u registry-pusher -p "$(oc create token registry-pusher -n ocp-poc --duration=1h)" "$REG"
> ```
>
> ✅ **Gerçek çıktı:** `Login Succeeded!`

> `--tls-verify=false`: bu route'un sertifikası bilinen bir CA tarafından imzalı değil (cluster-internal/self-signed CA). Production'da cluster CA'sını istemcinin trust store'una ekleyip bu flag'i kaldırmanız önerilir.

**2) Bir imajı namespace'inizin altına push edin:**

```bash
podman pull docker.io/library/alpine:3.20
podman tag docker.io/library/alpine:3.20 "$REG/ocp-poc/demo-alpine:latest"
podman push --tls-verify=false "$REG/ocp-poc/demo-alpine:latest"
```

> Push edebilmek için o namespace'te en az `edit` rolüne sahip olmanız yeterlidir.

**3) ImageStream'in otomatik oluştuğunu doğrulayın:**

```bash
oc get imagestream -n ocp-poc
oc describe imagestream demo-alpine -n ocp-poc
```

Push işlemi namespace'te otomatik olarak bir **ImageStream** (`demo-alpine`) oluşturur/günceller — ayrıca `oc create imagestream` çalıştırmanıza gerek yoktur.

**4) Cluster içinden, harici route'a hiç gerek olmadan bu imajı kullanın:**

```bash
oc run demo-alpine-test --image=image-registry.openshift-image-registry.svc:5000/ocp-poc/demo-alpine:latest --restart=Never -- sleep 3600
oc get pod demo-alpine-test
```

Cluster içi pod'lar internal registry'ye **service DNS** üzerinden erişir; kimlik doğrulaması pod'un service account token'ı ile otomatik yapılır — aynı namespace içindeyseniz ayrı bir `imagePullSecret` gerekmez.

**5) Kısa imaj adı kullanımı (image lookup policy):**

Varsayılan olarak bir Pod/Deployment'ta `image: demo-alpine:latest` gibi kısa bir isim yazamazsınız — tam internal registry pull spec'i gerekir. Bunu kolaylaştırmak için ImageStream'in **local lookup** politikasını açabilirsiniz:

```bash
oc set image-lookup demo-alpine
oc run demo-alpine-short --image=demo-alpine:latest --restart=Never -- sleep 3600
oc get pod demo-alpine-short -o jsonpath='{.spec.containers[0].image}'
# çıktı: image-registry.openshift-image-registry.svc:5000/ocp-poc/demo-alpine@sha256:...
```

OpenShift, bir admission webhook üzerinden kısa ismi otomatik olarak tam internal registry pull spec'ine (digest ile birlikte) çözümler.

**6) Harici olarak (cluster dışından) pull test:**

```bash
podman pull --tls-verify=false "$REG/ocp-poc/demo-alpine:latest"
```

✅ **Gerçek çıktı:** Push sonrası `demo-alpine` ImageStream'i `latest` tag'iyle otomatik oluştu. `demo-alpine-test` pod'u service DNS üzerinden çekip `Ready` oldu. `oc set image-lookup` sonrası kısa adla (`demo-alpine:latest`) oluşturulan pod'un image'ı `image-registry.openshift-image-registry.svc:5000/ocp-poc/demo-alpine@sha256:26ba97...` olarak çözümlendi. Yerel imaj silinip route üzerinden tekrar çekildi.

### Temizlik

```bash
oc delete imagestream demo-alpine -n ocp-poc
podman rmi -f docker.io/library/alpine:3.20 "$REG/ocp-poc/demo-alpine:latest"
podman logout "$REG"
```

## External Registry Entegrasyonu

Müşteri ortamında kullanılacak registry (Quay, Harbor, Artifactory, Nexus, Docker Hub, GHCR vb.) ne olursa olsun mekanizma aynıdır: public pull, private pull secret, ImageStream import. Bu bölüm bu mekanizmayı canlı olarak test edip belgeliyor; müşteri ortamında sadece registry host adı ve kimlik bilgileri değişir.

### Kavram

- **Public imajlar** için ekstra kimlik doğrulama gerekmez — herhangi bir external registry'den doğrudan pull edilebilir.
- **Private imajlar** için `kubernetes.io/dockerconfigjson` tipinde bir **Secret** oluşturup, ya pod'un `imagePullSecrets` alanına ya da namespace'in `default` service account'una bağlamanız gerekir.
- OpenShift ayrıca bir **ImageStream** ile harici registry'deki bir tag'i "import" ederek takip edebilir; `--scheduled=true` ile periyodik otomatik güncelleme (ve buna bağlı image-change trigger'ları) mümkündür.

### 1) Public external registry'den pull (canlı test edildi)

Quay.io üzerinden, kimlik doğrulama gerektirmeyen public bir imaj:

```bash
oc run quay-demo --image=quay.io/prometheus/busybox:latest --restart=Never -- sleep 3600
oc get pod quay-demo
```

Sonuç: `1/1 Running` — herhangi bir secret tanımlamadan doğrudan çalıştı.

### 2) Private registry için pull secret mekanizması (canlı test edildi)

Önce mekanizma **placeholder (geçersiz) kimlik bilgileriyle** gösterilir: OpenShift'in secret'ı okuyup registry'ye kimlik bilgisiyle bağlanmaya **çalıştığı** hata mesajının değişmesinden anlaşılır. Ardından (bölüm **e**) **geçerli** kimlik bilgileriyle başarılı private pull uçtan uca gösterilir.

**a) Secret olmadan bir private-tarzı imaj denemesi:**

```bash
oc run priv-nosecret --image=docker.io/ocppoc/private-demo:latest --restart=Never -- sleep 10
```

Gerçek sonuç (`oc describe pod`):

```
Failed to pull image "docker.io/ocppoc/private-demo:latest": ... requested access to the resource is denied
```

**b) `external-registry-pull-secret.yaml` ile secret oluşturup deneyin:**

```bash
oc apply -f external-registry-pull-secret.yaml
# veya imperative eşdeğeri:
oc create secret docker-registry external-registry-cred \
  --docker-server=docker.io \
  --docker-username=<gercek-kullanici-adi> \
  --docker-password=<gercek-sifre-veya-token> \
  --docker-email=<eposta>
```

Placeholder kimlik bilgileriyle tekrar denediğimizde **farklı** bir hata alınır — bu, secret'ın gerçekten okunup registry'ye gönderildiğinin kanıtıdır.

✅ **Gerçek çıktı (Docker Hub, 2026):**

```
Failed to pull image "docker.io/ocppoc/private-demo:latest": ... initializing source docker://ocppoc/private-demo:latest:
Requesting bearer token: received unexpected HTTP status: 400 Bad Request
```

(`requested access to the resource is denied` → kimlik bilgisi hiç gönderilmemiş; `Requesting bearer token ... 400` / `incorrect username or password` → kimlik bilgisi gönderilmiş ama **geçersiz**. Mesajın tam metni registry'ye ve zamana göre değişebilir.)

**c) İki bağlama yöntemi:**

- **Pod/Deployment seviyesinde** (yalnızca o workload için) — bkz. `external-image-demo.yaml`:
  ```yaml
  spec:
    imagePullSecrets:
      - name: external-registry-cred
  ```
- **Service account seviyesinde** (namespace'teki `default` SA'yı kullanan tüm pod'lar için otomatik, pod spec'inde ayrıca belirtmeye gerek kalmadan) — bu şekilde de test edildi ve aynı sonucu verdi:
  ```bash
  oc secrets link default external-registry-cred --for=pull
  ```

**d) Tam Deployment örneği:**

```bash
oc apply -f external-image-demo.yaml
oc get pods -l app=external-image-demo
# beklenen: ImagePullBackOff (placeholder kimlik bilgileri gerçek olmadığı için) —
# gerçek registry bilgileri girildiğinde Running olacaktır.
```

**e) Geçerli kimlik bilgisiyle private pull (uçtan uca):**

Gerçek bir private registry'yi temsil etmek için cluster'ın kendi registry'si **başka bir namespace'ten** çekilir. Bir namespace'teki imajı başka namespace'teki pod'lar kimlik bilgisi olmadan çekemez; yani private bir registry gibi davranır:

```bash
oc new-project ocp-poc-ext
IMG=image-registry.openshift-image-registry.svc:5000/ocp-poc/demo-alpine:latest

# 1) Secret olmadan
oc run nosecret --image=$IMG --restart=Never -n ocp-poc-ext -- sleep 3600

# 2) Sadece pull yetkisi olan bir ServiceAccount'un token'ı = "registry kullanıcı adı/şifresi"
oc create sa image-reader -n ocp-poc
oc policy add-role-to-user system:image-puller -z image-reader -n ocp-poc
oc create secret docker-registry private-registry-cred -n ocp-poc-ext \
  --docker-server=image-registry.openshift-image-registry.svc:5000 \
  --docker-username=image-reader --docker-password="$(oc create token image-reader -n ocp-poc --duration=8760h)"

# 3a) Pod seviyesinde (imagePullSecrets)  /  3b) ServiceAccount'a bağlayarak
oc secrets link default private-registry-cred --for=pull -n ocp-poc-ext
oc run sa-linked --image=$IMG --restart=Never -n ocp-poc-ext -- sleep 3600
```

✅ **Gerçek çıktı:**

| Durum | Sonuç |
|---|---|
| Secret yok | `Failed to pull image ...: authentication required` |
| `imagePullSecrets: [private-registry-cred]` olan pod | `Running` |
| Secret `default` SA'ya bağlı, pod spec'inde secret yok | `Running` |

Müşteri registry'sinde tek fark, `--docker-server/--docker-username/--docker-password` değerlerinin o registry'ye (örn. bir Quay robot hesabı) ait olmasıdır.

### 3) ImageStream ile harici imaj import/takip etme (canlı test edildi)

```bash
oc import-image external-busybox --from=quay.io/prometheus/busybox --confirm --scheduled=true
oc describe imagestream external-busybox
```

Gerçek çıktıda şu satır görülür:

```
latest
  updates automatically from registry quay.io/prometheus/busybox
```

`--scheduled=true` ile OpenShift, harici registry'yi periyodik olarak (varsayılan ~15 dakikada bir) kontrol eder; kaynak tag güncellenirse ImageStream otomatik güncellenir — bu da varsa bağlı bir `ImageChangeTrigger`'ı (örn. bir Build veya DeploymentConfig) tetikleyebilir.

### Temizlik

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

Araç: **Red Hat Advanced Cluster Security (RHACS / StackRox)**. ACS iki ana parçadan oluşur:

- **Central** — yönetim arayüzü, API, imaj/CVE veritabanı (`stackrox` namespace'inde çalışır).
- **Scanner** — imaj katmanlarını indeksleyip (Indexer) bilinen CVE veritabanıyla eşleştiren (Matcher) bileşen.

Ayrıca opsiyonel olarak her cluster'a bir **SecuredCluster** (Sensor + Collector + Admission Controller) kurularak runtime izleme ve deploy-time politika uygulaması (örn. "kritik CVE'si olan imaj deploy edilemesin") sağlanabilir — bu, salt imaj tarama için zorunlu değildir.

### Mevcut durumu kontrol etme

```bash
oc get ns | grep -iE "stackrox|rhacs"
oc get central -n stackrox
oc get securedcluster -n stackrox
oc get route central -n stackrox
```

| Bileşen | Beklenen durum |
|---|---|
| **Central** | `AVAILABLE: True` — UI + API erişilebilir |
| **Scanner** | İlgili pod'lar `Running`, CVE veritabanı güncel |
| **SecuredCluster** (varsa) | `AVAILABLE: True` — değilse `oc get securedcluster <ad> -o yaml` ile `status.conditions` altındaki hata mesajına bakın |

Central'a henüz kurulu değilse, kurulum **RHACS Operator** (OperatorHub üzerinden) ile yapılır: önce bir `Central` CR (Central servisleri) oluşturulur, ardından her secure edilecek cluster için bir `SecuredCluster` CR uygulanır. Bu POC'de kurulum adımı kapsam dışıdır — mevcut/kurulacak bir Central kullanılacaktır.

### Görüntü taraması için ön koşul: Image Integration

> **SecuredCluster kuruluysa** ACS, cluster'ın pull secret'larından ve internal registry'den otomatik integration'lar oluşturur (`Autogenerated https://image-registry.openshift-image-registry.svc:5000 for cluster <ad>` gibi). Bu durumda internal registry için elle integration eklemek gerekmez. Aşağıdaki adım SecuredCluster'ın olmadığı ya da otomatik integration'ın yetmediği durumlar içindir.

Central, bir registry'deki imajları tarayabilmesi için o registry'nin **Image Integration** olarak tanımlı olmasını gerektirir — aksi halde tarama isteği `no matching image registries found` hatasıyla reddedilir. Mevcut entegrasyonları listeleme:

```bash
CENTRAL=$(oc get route central -n stackrox -o jsonpath='{.spec.host}')
PASS="$(oc get secret central-htpasswd -n stackrox -o jsonpath='{.data.password}' | base64 -d)"
curl -sk -u "admin:$PASS" "https://$CENTRAL/v1/imageintegrations" | python3 -m json.tool
```

Internal registry (veya kullanılacak external registry) listede yoksa `acs-image-integration.json` dosyasını kendi kullanıcı adı/token bilgilerinizle doldurup ekleyin:

```bash
curl -sk -u "admin:$PASS" -X POST "https://$CENTRAL/v1/imageintegrations" \
  -H "Content-Type: application/json" \
  -d @acs-image-integration.json
```

> Demo/kurulum sırasında kişisel bir `oc whoami -t` token'ı kullanılabilir, ancak **production'da** bunun yerine sadece `system:image-puller` yetkisine sahip **dedicated bir service account token**'ı kullanılması önerilir (kişisel token'lar süreli/iptal edilebilir olduğu için entegrasyonu kırılgan yapar).

### `roxctl` ile imaj tarama

`roxctl` yoksa Central'ın kendisinden indirilebilir (sürüm Central ile birebir aynı olur):

```bash
curl -sk -u "admin:$PASS" -o roxctl "https://$CENTRAL/api/cli/download/roxctl-linux" && chmod +x roxctl
export ROX_ADMIN_PASSWORD="$PASS"
```

```bash
roxctl image scan \
  -e "${CENTRAL}:443" --insecure-skip-tls-verify \
  -p "$PASS" \
  --image image-registry.openshift-image-registry.svc:5000/<namespace>/<imaj>:<tag> \
  --output table
```

> `--insecure-skip-tls-verify`: route sertifikası self-signed/cluster-internal bir CA'ya aitse gereklidir. Route üzerinden büyük/yavaş taramalarda HAProxy'nin varsayılan idle timeout'u bağlantıyı kesebilir (`transport: EOF` hatası); bu durumda `oc port-forward svc/central 18443:443 -n stackrox` ile Central'a doğrudan bağlanıp `-e localhost:18443` kullanmak timeout'u aşar.

Önbelleğe alınmış eski bir sonuç yerine yeniden tarama için `--force` eklenir.

✅ **Gerçek çıktı:**

| İmaj | Süre | Sonuç |
|---|---|---|
| `registry.access.redhat.com/ubi8/ubi:8.4` (eski, bilinçli olarak yamasız) | 22 sn | `TOTAL-COMPONENTS: 112, TOTAL-VULNERABILITIES: 779, LOW: 285, MODERATE: 411, IMPORTANT: 83, CRITICAL: 0` |
| `quay.io/prometheus/busybox:latest` | <1 sn | `TOTAL-COMPONENTS: 0` (tek statik binary; paket veritabanı yok, beklenen) |
| `alpine:3.20` (Docker Hub ve internal registry) | <5 sn | `TOTAL-COMPONENTS: 0` — beklenmeyen; aşağıdaki kontrol sırasına bakın |

Lab ortamında Alpine için index raporu oluştu ama sonuç boş döndü. Ortamda hem `Scanner V4` hem `StackRox Scanner` integration'ı tanımlıydı; bu, aşağıdaki 3. maddedeki duruma uyuyor. Kurumsal (UBI/RHEL tabanlı) imajlar sorunsuz tarandı.

### Sonuç boş dönerse: kontrol sırası

Bir tarama `TOTAL-COMPONENTS: 0` gibi boş sonuç döndürürse, sırayla şunlar kontrol edilmeli:

1. **Image Integration doğru mu?** — `curl .../v1/imageintegrations` çıktısında ilgili registry'nin `endpoint` alanı, taranan imajın registry host'uyla birebir eşleşmeli (örn. `docker.io` ile `registry-1.docker.io` farklı değerlerdir, eşleşmezse entegrasyon devreye girmez).
2. **Indexer imajı işledi mi?** — `oc logs -n stackrox deploy/scanner-v4-indexer --since=5m | grep "manifest successfully scanned"` ile katmanların okunduğunu doğrulayın.
3. **Matcher çağrıldı mı?** — `oc logs -n stackrox deploy/scanner-v4-matcher --since=5m | grep GetVulnerabilities` ile CVE eşleştirme adımının tetiklendiğini doğrulayın. Indexer başarılı olduğu halde Matcher hiç çağrılmıyorsa, bu Central'ın kendi enrichment pipeline'ında bir sorun olduğuna işaret eder (örn. birden fazla scanner entegrasyonunun çakışması) — bu durumda ACS'i işleten platform/operatör ekibiyle iletişime geçilmesi gerekir.
4. **İmaj Central'da kayıtlı mı?** — `curl .../v1/images?query=Image:<imaj-adi>` ile imajın Central'ın kendi veritabanına işlenip işlenmediğini kontrol edin.

### Temizlik

```bash
oc delete imagestream <test-imaji> -n <namespace> --ignore-not-found
podman rmi -f <yerel-imaj-referanslari>
podman logout "$REG"
```
