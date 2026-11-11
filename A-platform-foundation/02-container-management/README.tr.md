# 02 — Container Yönetimi

> 🇬🇧 [English](README.md) · [← 01 — Kimlik ve Yetki](../01-identity-and-access/README.tr.md) · [POC akışı](../../README.tr.md) · [03 — Uygulama Senaryoları →](../03-application-scenarios/README.tr.md)

Bu bölümde OpenShift üzerinde container/pod yönetimi ile ilgili aşağıdaki konular ele alınacaktır. Tüm adımlar lab ortamında (OpenShift 4.22) **uçtan uca canlı test edilmiştir**; "✅ Gerçek çıktı" satırları bu testlerden alınmıştır.

- [x] Namespace/Project yapısı
- [x] Deployment rollout/rollback
- [x] Horizontal Pod Autoscaler
- [x] Resource Quota
- [x] LimitRange

---

## Namespace/Project Yapısı

### Namespace nedir?

Namespace, bir Kubernetes/OpenShift cluster'ı içindeki kaynakları (Pod, Service, Deployment, ConfigMap vb.) mantıksal olarak birbirinden izole eden bir bölümlendirme mekanizmasıdır. Aynı cluster üzerinde farklı takımların, ortamların (dev/test/prod) veya uygulamaların kaynakları birbirine karışmadan, isim çakışması yaşamadan çalışabilmesini sağlar. Kaynak kotaları (ResourceQuota), yetkilendirme (RBAC) ve ağ politikaları (NetworkPolicy) genellikle namespace bazında tanımlanır.

### Project (Proje) nedir?

Project, OpenShift'in Kubernetes Namespace kavramını genişleterek üzerine eklediği kendine özgü bir kaynaktır (`project.openshift.io/v1`). Bir Project oluşturulduğunda arka planda otomatik olarak aynı isimde bir Namespace de oluşur. Project; Namespace'e ek olarak varsayılan RBAC rollerinin (admin, edit, view) otomatik atanması, `display-name` ve `description` gibi ek bilgiler ve OpenShift Web Console'da özel bir görünüm sağlar.

**Özetle:** Her Project bir Namespace'tir, ancak her Namespace bir Project değildir (düz `oc apply -f namespace.yaml` ile oluşturulan bir Namespace, `oc new-project` ile oluşturulan kadar zengin varsayılan ayarlara sahip olmayabilir; ancak `openshift.io/display-name` ve `openshift.io/description` anotasyonları eklenirse Console'da Project gibi görünür).

### YAML Dosyası

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

### YAML nasıl apply edilir?

```bash
# Namespace/Project oluşturma veya güncelleme
oc apply -f namespace.yaml

# Doğrulama
oc get namespace ocp-poc
oc get project ocp-poc
oc describe project ocp-poc
```

`oc apply` deklaratif bir komuttur: YAML içindeki tanım cluster'daki mevcut durumla karşılaştırılır, kaynak yoksa oluşturulur, varsa fark kadar güncellenir. Bu nedenle tekrar tekrar çalıştırmak güvenlidir (idempotent).

### `oc` komutu ile, apply kullanmadan yeni proje nasıl oluşturulur?

YAML dosyasına gerek kalmadan, doğrudan komut satırından yeni bir proje oluşturmak için:

```bash
oc new-project ocp-poc \
  --display-name="OCP POC" \
  --description="Namespace/project for the OpenShift POC"
```

Bu komut hem Namespace/Project'i oluşturur hem de çalıştığınız context'i (`oc project`) otomatik olarak yeni oluşturulan projeye geçirir. Var olan bir projeye geçmek için:

```bash
oc project ocp-poc
```

### GUI (Web Console) üzerinden proje nasıl oluşturulur?

1. OpenShift Web Console'a giriş yapın.
2. Sol üstten **Developer** görünümüne geçin (veya **Administrator** görünümünde kalabilirsiniz).
3. Sol menüden **Home > Projects** sekmesine gidin.
4. Sağ üstteki **Create Project** butonuna tıklayın.
5. Açılan formda:
   - **Name**: `ocp-poc`
   - **Display Name**: `OCP POC`
   - **Description**: `Namespace/project for the OpenShift POC`
6. **Create** butonuna basarak projeyi oluşturun.
7. Oluşturduktan sonra üst menüden proje seçici (project selector) ile bu proje arasında geçiş yapabilirsiniz.

### Namespace/Project nasıl silinir?

**CLI ile:**

```bash
# Project olarak silme (önerilen, OpenShift'e özgü kaynak)
oc delete project ocp-poc

# veya düz Namespace olarak silme
oc delete namespace ocp-poc

# YAML dosyası üzerinden silme
oc delete -f namespace.yaml
```

> Not: Silme işlemi geri alınamaz ve namespace içindeki **tüm** kaynaklar (Pod, Deployment, Service, ConfigMap, Secret vb.) birlikte silinir. Üretim ortamlarında dikkatli kullanılmalıdır.

**GUI ile:**

1. Web Console'da **Home > Projects** sekmesine gidin.
2. Silinecek projenin (`ocp-poc`) satırındaki `⋮` (üç nokta) menüsüne tıklayın.
3. **Delete Project** seçeneğini seçin.
4. Açılan onay penceresine proje adını (`ocp-poc`) yazarak silme işlemini onaylayın.

---

## Deployment rollout/rollback

Bu bölümde `nginx.yaml` dosyası ile Deployment, ConfigMap, Secret, Service ve Route kaynaklarından oluşan basit bir Nginx uygulaması `ocp-poc` namespace'ine kurulacak. Ardından bu Deployment üzerinde `oc rollout` komutlarıyla rollout/rollback senaryosu gösterilecektir.

### nginx.yaml içeriği

- **Secret (`nginx-conf`)** — Nginx'in `conf.d/default.conf` dosyasını `stringData` içinde tutar. Container 8080 portunda unprivileged olarak çalışacağı için özel bir server bloğu tanımlanmıştır.
- **ConfigMap (`nginx-index-html`)** — `index.html` statik sayfasını tutar.
- **Deployment (`nginx`)** — `nginxinc/nginx-unprivileged:1.27-alpine` imajını kullanan, 2 replikalı, ConfigMap ve Secret'i volume olarak mount eden pod şablonu.
- **Service (`nginx`)** — Deployment'i cluster içinde 8080 portundan expose eder.
- **Route (`nginx`)** — Service'i cluster dışına HTTP olarak açar (host belirtilmediği için OpenShift otomatik bir hostname üretir).

> **Neden `nginx-unprivileged` ve 8080?** Standart `nginx` imajı OpenShift'in varsayılan **restricted** SCC'si altında (rastgele/non-root UID, hiçbir capability yok) çalışırken sorun çıkarır: hem 80 gibi ayrıcalıklı bir porta bind edemez (`NET_BIND_SERVICE` capability'si yok) hem de `/var/cache/nginx` gibi dizinlere yazma izni bulamaz. `nginxinc/nginx-unprivileged` imajı bu senaryo için tasarlanmıştır ve 8080 portunda dinler; bu yüzden bu POC'de tercih edilmiştir.

İlgili `index.html` ve `nginx.conf` dosyaları, aşağıdaki `oc create ... --from-file` örneklerinde kullanılmak üzere aynı dizine ayrıca düz dosya olarak da eklendi (`index.html`, `nginx.conf`).

### `oc apply` ile oluşturma

```bash
oc project ocp-poc   # doğru namespace'te olduğumuzdan emin olalım
oc apply -f nginx.yaml
```

Doğrulama:

```bash
oc get deployment,configmap,secret,svc,route -l app=nginx
oc get pods -l app=nginx -w
oc get route nginx -o jsonpath='{.spec.host}'
```

Route'un ürettiği hostname'e `curl` ile gidildiğinde `index.html` içeriği ve `nginx.conf`'ta eklenen özel `X-Served-By` header'ı görülebilir:

```bash
curl -I http://$(oc get route nginx -o jsonpath='{.spec.host}')
```

✅ **Gerçek çıktı:** Deployment `2/2` hazır; `curl -I` → `HTTP/1.1 200 OK`, `x-served-by: ocp-poc-nginx`; sayfa gövdesinde `<h1>OpenShift POC</h1>`. Aşağıdaki imperative komutlarla ayrı bir namespace'te kurulan aynı uygulama da aynı cevabı verdi.

### Aynı kaynakların imperative (`oc create` / `oc expose`) komutlarıyla oluşturulması

`nginx.yaml` yerine, aynı kaynaklar tek tek imperative komutlarla da oluşturulabilir:

```bash
# ConfigMap - index.html dosyasından
oc create configmap nginx-index-html --from-file=index.html=index.html

# Secret - nginx.conf dosyasından
oc create secret generic nginx-conf --from-file=nginx.conf=nginx.conf

# Deployment - imaj üzerinden
oc create deployment nginx --image=nginxinc/nginx-unprivileged:1.27-alpine --replicas=2

# Deployment'a ConfigMap'i volume olarak eklemek
oc set volume deployment/nginx --add --name=index-html \
  --type=configmap --configmap-name=nginx-index-html \
  --mount-path=/usr/share/nginx/html/index.html --sub-path=index.html

# Deployment'a Secret'i volume olarak eklemek
oc set volume deployment/nginx --add --name=nginx-conf \
  --type=secret --secret-name=nginx-conf \
  --mount-path=/etc/nginx/conf.d/default.conf --sub-path=nginx.conf

# Deployment'ı cluster içi bir Service olarak dışarı açmak
oc expose deployment nginx --port=8080 --target-port=8080 --name=nginx

# Service'i Route ile cluster dışına açmak
oc expose svc/nginx
```

> `oc create deployment` ile oluşturulan Deployment; YAML'daki gibi readiness/liveness probe veya resource limit gibi detayları içermez. Bu detaylar sonradan `oc set probe`, `oc set resources` komutlarıyla ya da `oc edit deployment nginx` ile eklenebilir. Gerçek projelerde deklaratif YAML (GitOps) tercih edilir; imperative komutlar hızlı prototipleme/öğrenme amaçlıdır.

### Rollout / Rollback Senaryosu

`oc rollout` komutları bir Deployment'ın sürüm geçmişini yönetmek için kullanılır.

**1) Mevcut durumu ve geçmişi görüntüleme**

```bash
oc rollout status deployment/nginx
oc rollout history deployment/nginx
```

**2) Yeni (ve kasıtlı olarak hatalı) bir sürüm yayınlamak — rollout tetiklemek**

```bash
# var olmayan bir image tag'i ile "bozuk" bir deploy simüle edelim
oc set image deployment/nginx nginx=nginxinc/nginx-unprivileged:this-tag-does-not-exist
```

**3) Rollout durumunu izleme**

```bash
oc rollout status deployment/nginx
# yeni pod'lar ImagePullBackOff / ErrImagePull durumuna düşecektir
oc get pods -l app=nginx
```

✅ **Gerçek çıktı:** `oc rollout status` → `error: timed out waiting for the condition`. Pod listesi: eski ReplicaSet'in iki pod'u `Running`, yeni pod `ImagePullBackOff`. Rolling update yeni pod hazır olmadığı için eski pod'ları kapatmadı; uygulama kesintisiz çalıştı.

**4) Sorunlu rollout'u geri almak (rollback)**

```bash
# bir önceki çalışan sürüme dön
oc rollout undo deployment/nginx

# belirli bir revizyona dönmek isterseniz:
oc rollout history deployment/nginx                  # revizyon numaralarını listeler
oc rollout history deployment/nginx --revision=2      # o revizyonun detayına bakar
oc rollout undo deployment/nginx --to-revision=2
```

✅ **Gerçek çıktı:** `deployment.apps/nginx rolled back` → `successfully rolled out`, image tekrar `nginxinc/nginx-unprivileged:1.27-alpine`.

**5) Birden fazla değişikliği tek rollout olarak yaymak: pause / resume**

```bash
oc rollout pause deployment/nginx
oc set image deployment/nginx nginx=nginxinc/nginx-unprivileged:1.27-alpine
oc set env deployment/nginx POC_SURUM=v2
oc rollout resume deployment/nginx   # pause sırasında biriken değişiklikler tek seferde uygulanır
```

✅ **Gerçek çıktı:** Pause sırasında yapılan değişiklikler pod'lara yansımadı. `resume` sonrası tek bir rollout ile yeni pod'lar `POC_SURUM=v2` ortam değişkeniyle açıldı.

> Bu adımda kaynak limitleri (`oc set resources`) **değiştirilmez**: aşağıdaki Resource Quota bölümündeki hesaplar nginx'in `nginx.yaml`'daki değerleriyle (limits 200m/128Mi) çalıştığını varsayar.

**6) Pod'ları image değişmeden yeniden başlatmak (restart)**

```bash
oc rollout restart deployment/nginx
```

> **Önemli not:** `index.html` veya `nginx.conf` içeriğini güncelleyip tekrar `oc apply -f nginx.yaml` çalıştırdığınızda, Deployment'ın pod template'i (image, env, vb.) değişmediği için **otomatik olarak yeni bir rollout tetiklenmez** — mevcut pod'lar eski ConfigMap/Secret içeriğiyle çalışmaya devam eder. Yeni içeriğin pod'lara yansıması için `oc rollout restart deployment/nginx` çalıştırılmalıdır.

## Horizontal Pod Autoscaler

### HPA nedir?

HorizontalPodAutoscaler (HPA), bir Deployment/StatefulSet gibi bir workload'ın pod (replika) sayısını, gözlemlenen metriklere (varsayılan olarak CPU/memory kullanımı, isteğe bağlı custom/external metrikler) göre otomatik olarak artırıp azaltan bir Kubernetes/OpenShift kaynağıdır. HPA controller periyodik olarak (varsayılan 15 saniyede bir) hedef metriği ölçer ve kabaca şu formülle istenen replika sayısını hesaplar:

```
istenenReplika = ceil( mevcutReplika × ( mevcutMetrikDeğeri / hedefMetrikDeğeri ) )
```

CPU tabanlı bir HPA'nın çalışabilmesi için hedeflenen container'da **mutlaka `resources.requests.cpu` tanımlı olmalıdır** — utilization yüzdesi bu request değerine göre hesaplanır.

### Demo senaryosu: `php-apache`

nginx örneğindeki gibi CPU tüketimini "doğal" bir istekle tetiklemeye çalışmak yerine (statik içerik servis eden bir web sunucusunun CPU'sunu anlamlı şekilde zorlamak zordur ve sonucu öngörülemez), Kubernetes/OpenShift'in resmi HPA dokümantasyonunda kullanılan **`php-apache`** demo imajını kullanıyoruz. Bu imaj gelen her isteğe karşılık kasıtlı olarak CPU yoğun bir hesaplama (sqrt döngüsü) yapar; bu sayede yük altında CPU kullanımı net ve öngörülebilir şekilde yükselir, HPA'nın scale-up/scale-down davranışı açıkça gözlemlenebilir.

`hpa.yaml` içeriği:

- **Deployment (`php-apache`)** — `registry.k8s.io/hpa-example` imajı, `requests.cpu: 200m` / `limits.cpu: 500m`.
- **Service (`php-apache`)** — cluster içi erişim için ClusterIP, port 80.
- **Route (`php-apache`)** — dışarıdan da tarayıcıyla test edebilmek için (opsiyonel, yük testi için gerekli değil).
- **HorizontalPodAutoscaler (`php-apache`)** — `minReplicas: 1`, `maxReplicas: 5`, hedef CPU utilization `%50`. Demoyu makul sürede gözlemleyebilmek için `behavior.scaleDown.stabilizationWindowSeconds` **60 saniyeye** düşürüldü (varsayılan değer 300 saniyedir — production'da ani düşüşlerde "flapping"i önlemek için genelde varsayılan/daha yüksek tutulur).

> **SCC / port 80 notu:** `hpa-example` imajı klasik bir Apache/root imajıdır ve OpenShift'in varsayılan **restricted** SCC'si altında (rastgele UID, `NET_BIND_SERVICE` capability'si yok) port 80'e bind olamaz.
>
> ✅ **Gerçek çıktı:** Pod `restricted-v2` SCC'siyle `Error` durumuna düştü: `(13)Permission denied: AH00072: make_sock: could not bind to address 0.0.0.0:80`, `no listening sockets available, shutting down`.
>
> Bunu çözmek için bu namespace'in `default` service account'una `anyuid` SCC'si verilir (cluster-admin yetkisi ister). Sonrasında pod `scc=anyuid` ile `Running` oldu:
>
> ```bash
> oc adm policy add-scc-to-user anyuid -z default -n ocp-poc
> ```
>
> Bu, sadece demo/POC amaçlıdır; gerçek projelerde imajın OpenShift-uyumlu (arbitrary UID'ye hazır) şekilde yeniden derlenmesi tercih edilir — bunu ileride **[04 — Image Yönetimi](../04-image-management/README.tr.md)** ve **[05 — CI/CD](../../B-application-delivery/05-ci-cd/README.tr.md)** rehberlerinde ele alacağız.

### `oc apply` ile oluşturma

```bash
oc project ocp-poc
oc apply -f hpa.yaml

# gerekiyorsa (bkz. yukarıdaki SCC notu):
oc adm policy add-scc-to-user anyuid -z default -n ocp-poc
oc rollout restart deployment/php-apache
```

Doğrulama:

```bash
oc get deployment,svc,route,hpa -l app=php-apache
oc describe hpa php-apache
```

### İmperative eşdeğer: `oc autoscale`

`hpa.yaml` içindeki HPA kaynağı, deployment zaten varken tek komutla da oluşturulabilir:

```bash
oc create deployment php-apache --image=registry.k8s.io/hpa-example
oc set resources deployment/php-apache -c php-apache --requests=cpu=200m,memory=128Mi --limits=cpu=500m,memory=256Mi
oc expose deployment php-apache --port=80
oc expose svc/php-apache

oc autoscale deployment/php-apache --cpu-percent=50 --min=1 --max=5
```

### GUI (Web Console) üzerinden HPA oluşturma

1. **Developer** görünümünde **Topology**'ye gidin, `php-apache` Deployment'ına tıklayın.
2. Sağdaki panelden **Actions > Add HorizontalPodAutoscaler** seçin (veya **Administrator** görünümünde **Workloads > HorizontalPodAutoscalers > Create HorizontalPodAutoscaler**).
3. **Minimum/Maximum Pods**: `1` / `5`, **CPU Utilization**: hedef `%50` olarak girin.
4. **Create** ile kaydedin. Oluşan HPA, **Workloads > HorizontalPodAutoscalers** altında listelenir ve mevcut/hedef metrik değerleri buradan izlenebilir.

### Scale-up / Scale-down Senaryosu

**1) Başlangıç durumunu gözlemleyin** (tek terminalde canlı izleyin):

```bash
oc get hpa php-apache -w
```

İkinci bir terminalde pod'ları izleyin:

```bash
oc get pods -l app=php-apache -w
```

**2) Yük üretin** — `load-generator.yaml` dosyasındaki Job, 4 paralel pod ile `php-apache` servisine sürekli istek gönderir:

```bash
oc apply -f load-generator.yaml
oc get pods -l app=load-generator
```

> Alternatif (tek seferlik, kalıcı kaynak bırakmayan) yöntem: `oc run -i --tty load-generator --rm --image=busybox:1.36 --restart=Never -- /bin/sh -c "while true; do wget -q -O- http://php-apache; done"`

**3) Scale-up'ı izleyin** — 4 paralel load-generator pod'u yeterince istek ürettiği için yaklaşık **1 dakika içinde** CPU kullanımı `%50` hedefinin çok üzerine çıkar ve HPA `maxReplicas: 5`'e hızla ulaşır:

✅ **Gerçek çıktı (`oc get hpa`, yük başladıktan sonra):**

```
 31s  TARGETS=56%/50%   REPLICAS=1
 52s  TARGETS=249%/50%  REPLICAS=2
 62s  TARGETS=249%/50%  REPLICAS=5
124s  TARGETS=124%/50%  REPLICAS=5     # 5 pod'a dağılınca pod başına kullanım düştü
```

```bash
oc describe hpa php-apache   # Events kısmında "SuccessfulRescale" mesajlarını gösterir
```

**4) Yükü durdurun** ve scale-down'ı izleyin:

```bash
oc delete -f load-generator.yaml
```

CPU kullanımı düştükten sonra HPA önce tanımladığımız `stabilizationWindowSeconds: 60` kadar bekler, ardından `behavior.scaleDown.policies` içindeki `Type: Pods, Value: 1, Period: 60s` kuralı gereği **her 60 saniyede en fazla 1 pod** azaltarak `minReplicas: 1`'e iner. Yani `maxReplicas: 5`'ten `1`'e tam inme toplamda **~4-5 dakika** sürer. `oc get hpa php-apache -w` ile bu düşüşü canlı izleyebilirsiniz.

✅ **Gerçek çıktı (yük durdurulduktan sonra):** CPU ~85. saniyede `%12`'ye, ~96. saniyede `%0`'a düştü. Replikalar **5 → 4 (138 sn) → 3 (201 sn) → 2 (264 sn) → 1 (327 sn)**, yani dakikada bir pod azaldı. `oc describe hpa` Events: `New size: 4; reason: All metrics below target` ... `New size: 1`.

### HPA nasıl silinir?

**CLI ile:**

```bash
oc delete hpa php-apache
# veya tüm demo kaynaklarını birlikte kaldırmak için:
oc delete -f hpa.yaml
```

**GUI ile:**

1. **Administrator** görünümünde **Workloads > HorizontalPodAutoscalers**'a gidin.
2. `php-apache` satırındaki `⋮` menüsünden **Delete HorizontalPodAutoscaler**'ı seçin ve onaylayın.

> Not: HPA'yı silmek, altındaki Deployment'ı veya mevcut pod'ları silmez; sadece otomatik ölçeklendirmeyi durdurur, replika sayısı silme anındaki son değerde sabit kalır.

## Resource Quota

### ResourceQuota nedir?

ResourceQuota, bir namespace içindeki **toplam** kaynak tüketimini (CPU, memory, pod/configmap/secret/service/PVC sayısı vb.) üst sınırlarla kısıtlayan bir kaynaktır. HPA veya manuel scale ile pod sayısı artsa bile, namespace bu sınırları aşacak yeni bir pod'un **oluşturulmasına izin vermez** — Deployment/ReplicaSet o pod'u oluşturmaya çalışır ama admission aşamasında reddedilir ve pod `Pending`/oluşturulamamış kalır.

> **Önemli:** Bir ResourceQuota'da `requests.cpu`/`requests.memory`/`limits.cpu`/`limits.memory` tanımlıysa, o namespace'teki **her** pod artık bu alanları açıkça belirtmek zorundadır; aksi halde pod oluşturma isteği "must specify limits.cpu" gibi bir hatayla reddedilir. Bu yüzden ResourceQuota, aşağıdaki **LimitRange** ile birlikte kullanılır: LimitRange, değer belirtilmeyen container'lara otomatik varsayılan request/limit atayarak bu zorunluluğu kullanıcıdan gizler.

`resourcequota.yaml` içeriği — `ocp-poc` namespace'i için:

| Alan | Değer | Açıklama |
|---|---|---|
| `requests.cpu` | 500m | Namespace'teki tüm pod'ların CPU **request** toplamı |
| `requests.memory` | 512Mi | Namespace'teki tüm pod'ların memory **request** toplamı |
| `limits.cpu` | 1 | Tüm pod'ların CPU **limit** toplamı |
| `limits.memory` | 1Gi | Tüm pod'ların memory **limit** toplamı |
| `pods` | 10 | Aynı anda çalışabilecek maksimum pod sayısı |
| `configmaps`, `secrets`, `services`, `persistentvolumeclaims` | 20/20/10/5 | İlgili kaynak sayısı üst sınırları |

### `oc apply` ile oluşturma

```bash
oc project ocp-poc
oc apply -f resourcequota.yaml
oc describe resourcequota ocp-poc-quota
```

Bu noktada, önceki bölümlerde oluşturduğumuz `nginx` (2 replika, requests 50m/64Mi, limits 200m/128Mi) ve `php-apache` (1 replika, requests 200m/128Mi, limits 500m/256Mi) deployment'ları zaten çalışıyorsa `oc describe resourcequota` çıktısında şu **Used** değerlerini görürsünüz — hepsi hâlâ hard limitlerin altında olduğu için mevcut pod'lar etkilenmez (✅ canlı testte birebir bu değerler alındı):

```
Resource            Used   Hard
--------            ----   ----
requests.cpu        300m   500m
requests.memory     256Mi  512Mi
limits.cpu          900m   1
limits.memory       512Mi  1Gi
pods                3      10
```

### İmperative eşdeğer: `oc create quota`

```bash
oc create quota ocp-poc-quota \
  --hard=requests.cpu=500m,requests.memory=512Mi,limits.cpu=1,limits.memory=1Gi,pods=10
```

Sayı kotaları (`configmaps`, `secrets`, `services`, `persistentvolumeclaims`) da aynı `--hard` listesine eklenebilir: `--hard=...,configmaps=20,secrets=20,services=10,persistentvolumeclaims=5` (canlı testte doğrulandı). Yine de kalıcı tanımlar için YAML (GitOps) tercih edilir.

### Senaryo: Quota'yı aşmak

Yukarıdaki tabloda `limits.cpu` zaten **900m/1000m** (%90) seviyesinde — her yeni nginx pod'u 200m ek `limits.cpu` istediği için, replika sayısını sadece **3**'e çıkarmak bile quota'yı aşmaya yeter (900m + 200m = 1100m > 1000m hard):

```bash
oc scale deployment/nginx --replicas=3
oc get replicaset -l app=nginx
oc describe replicaset <yeni-nginx-replicaset-adı>
```

`describe` çıktısının **Events** kısmında şuna benzer bir hata görürsünüz:

```
Warning  FailedCreate  ...  Error creating: pods "nginx-xxxxxxxxxx-yyyyy" is forbidden:
exceeded quota: ocp-poc-quota, requested: limits.cpu=200m,
used: limits.cpu=900m, limited: limits.cpu=1
```

✅ **Gerçek çıktı:** ReplicaSet `desired=3 current=2 ready=2`; Event: `pods "nginx-...-6gwgs" is forbidden: exceeded quota: ocp-poc-quota, requested: limits.cpu=200m, used: limits.cpu=900m, limited: limits.cpu=1`.

Deployment, eski (sağlıklı) ReplicaSet'i koruyarak çalışmaya devam eder — 3. pod hiç oluşturulamadığı için servis kesintisi yaşanmaz. Doğrulama ve geri alma:

```bash
oc describe resourcequota ocp-poc-quota   # Used değişmediğini gösterir
oc scale deployment/nginx --replicas=2       # eski duruma dön
```

### ResourceQuota nasıl silinir?

```bash
oc delete resourcequota ocp-poc-quota
# veya
oc delete -f resourcequota.yaml
```

GUI'de: **Administrator** görünümü → **Administration > ResourceQuotas** → satırdaki `⋮` menüsü → **Delete ResourceQuota**.

## LimitRange

### LimitRange nedir?

LimitRange, ResourceQuota'nın aksine **tek tek** Pod/Container/PVC seviyesinde sınır koyar: bir container'ın alabileceği minimum/maksimum CPU-memory değerlerini, ve değer belirtilmediğinde uygulanacak **varsayılan** (`default`) ve **varsayılan request** (`defaultRequest`) değerlerini tanımlar.

`limitrange.yaml` içeriği — `ocp-poc-limits`:

| Kapsam | Alan | Değer |
|---|---|---|
| Container | `defaultRequest` (cpu/memory) | 50m / 64Mi |
| Container | `default` (cpu/memory limit) | 100m / 128Mi |
| Container | `min` | 25m / 32Mi |
| Container | `max` | 500m / 512Mi |
| Pod | `max` (toplam) | 1 cpu / 1Gi |
| PersistentVolumeClaim | `min` / `max` | 1Gi / 10Gi |

### `oc apply` ile oluşturma

```bash
oc apply -f limitrange.yaml
oc describe limitrange ocp-poc-limits
```

> **İmperative eşdeğer yok:** `oc create`/`kubectl create` altında `configmap`, `secret`, `quota` gibi kısayolların aksine **LimitRange için imperative bir alt komut bulunmaz**. LimitRange yalnızca `oc apply -f` (veya `oc create -f`) ile bir YAML/JSON manifestosu üzerinden oluşturulabilir.

### Senaryo 1: Varsayılan değerlerin otomatik atanması

Kaynak belirtmeden geçici bir pod oluşturalım:

```bash
oc run limit-test --image=busybox:1.36 --restart=Never -- sleep 3600
```

Pod'un aldığı resources'a bakalım — hiçbir şey belirtmediğimiz halde LimitRange'in `defaultRequest`/`default` değerlerini otomatik enjekte ettiğini göreceksiniz:

```bash
oc get pod limit-test -o jsonpath='{.spec.containers[0].resources}{"\n"}'
# ✅ Gerçek çıktı: {"limits":{"cpu":"100m","memory":"128Mi"},"requests":{"cpu":"50m","memory":"64Mi"}}

oc delete pod limit-test
```

### Senaryo 2: `max` sınırının aşılması (rollout ile birleşik senaryo)

nginx container'ının limitini, LimitRange'in izin verdiği `max.cpu: 500m` değerinin üzerine çıkarmayı deneyelim:

```bash
oc set resources deployment/nginx -c nginx --limits=cpu=800m,memory=512Mi
oc get replicaset -l app=nginx
oc describe replicaset <yeni-nginx-replicaset-adı>
```

Events kısmında admission reddi görünür (✅ canlı testte birebir bu mesaj alındı; nginx `2/2` hazır kalmaya devam etti):

```
Warning  FailedCreate  ...  Error creating: pods "nginx-xxxxxxxxxx-zzzzz" is forbidden:
maximum cpu usage per Container is 500m, but limit is 800m
```

Bu, **Deployment rollout/rollback** bölümünde gördüğümüz mekanizmanın aynısıdır: yeni ReplicaSet sağlıklı pod oluşturamadığı için Deployment eski ReplicaSet üzerinde çalışmaya devam eder, servis kesintiye uğramaz. Geri almak için:

```bash
oc set resources deployment/nginx -c nginx \
  --requests=cpu=50m,memory=64Mi --limits=cpu=200m,memory=128Mi
```

(İstersek `oc rollout undo deployment/nginx` ile de eski ReplicaSet'e dönebiliriz — ancak burada eski ReplicaSet zaten aktif kaldığı için ek bir işlem gerekmez.)

### `min` sınırının aşılması (kısaca)

Aynı mantıkla, `min.cpu: 25m` altında bir değer istemek de reddedilir:

```bash
oc set resources deployment/nginx -c nginx --requests=cpu=10m
# ✅ Gerçek çıktı: pods "..." is forbidden: minimum cpu usage per Container is 25m, but request is 10m
```

### LimitRange nasıl silinir?

```bash
oc delete limitrange ocp-poc-limits
# veya
oc delete -f limitrange.yaml
```

GUI'de: **Administrator** görünümü → **Administration > LimitRanges** → satırdaki `⋮` menüsü → **Delete LimitRange**.

> Not: LimitRange silindiğinde, halihazırda oluşturulmuş pod'ların resources değerleri **değişmez** (zaten yaratılırken enjekte edilmişti); sadece bundan sonra oluşturulacak yeni pod'lar için varsayılan/min/max kontrolü uygulanmaz.

---

## Sonraki Rehbere Geçmeden

`ocp-poc` namespace'i ve `nginx` uygulaması **[03 — Uygulama Senaryoları](../03-application-scenarios/README.tr.md)** rehberinde kullanılmaya devam eder. Ancak bu rehberde oluşturulan **ResourceQuota ve LimitRange açık kalırsa** 02'deki PostgreSQL pod'u oluşturulamaz.

✅ **Gerçek çıktı (kota kaldırılmadan 03'e geçildiğinde):** `Create Pod postgres-0 in StatefulSet postgres failed error: pods "postgres-0" is forbidden: exceeded quota: ocp-poc-quota, requested: limits.cpu=300m, used: limits.cpu=900m, limited: limits.cpu=1`

Bu yüzden 03'e geçmeden HPA demosunu, kotayı ve limitrange'i kaldırın; `anyuid` yetkisini de geri alın:

```bash
oc delete -f hpa.yaml -f resourcequota.yaml -f limitrange.yaml
oc adm policy remove-scc-from-user anyuid -z default -n ocp-poc
```

Bu adımdan sonra (kota yüzünden bekleyen) StatefulSet kendiliğinden toparlanır: canlı testte `postgres-0` 52 sn içinde `Running` oldu.

## Temizlik (POC sonunda)

```bash
oc delete project ocp-poc
```
