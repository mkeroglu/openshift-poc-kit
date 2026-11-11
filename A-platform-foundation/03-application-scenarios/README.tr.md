# 03 — Uygulama Senaryoları

> 🇬🇧 [English](README.md) · [← 02 — Container Yönetimi](../02-container-management/README.tr.md) · [POC akışı](../../README.tr.md) · [04 — Image Yönetimi →](../04-image-management/README.tr.md)

Bu bölümde OpenShift üzerinde farklı uygulama karakteristiklerinin (stateless, stateful, API/microservice) nasıl modellendiği ele alınacaktır. Tüm örnekler `ocp-poc` namespace'inde çalışır (bkz. `../02-container-management/namespace.yaml`). Tüm adımlar lab ortamında (OpenShift 4.22) **uçtan uca canlı test edilmiştir**.

> **Ön koşul:** 02'de oluşturulan ResourceQuota/LimitRange bu rehberden önce kaldırılmış olmalıdır (bkz. [02 — Container Yönetimi](../02-container-management/README.tr.md) → "Sonraki Rehbere Geçmeden"). Aksi halde `postgres-0` kota yüzünden oluşturulamaz.

- [x] Stateless web app
- [x] Stateful database app
- [x] API/microservice app

---

## Stateless Web App

### Stateless nedir?

Bir uygulamanın **stateless** olması, işlediği isteklerle ilgili kalıcı bir durumu (state) kendi içinde (diskinde/belleğinde) tutmaması anlamına gelir. Pratikte şu özellikler aranır:

- Herhangi bir replika/pod, diğerinin birebir yerine geçebilir (**interchangeable**) — hangi pod'un isteği karşıladığının önemi yoktur.
- Pod'un silinip yeniden oluşturulması (kaza, node tahliyesi, rolling update) **veri kaybına yol açmaz**, çünkü zaten kalıcı bir veri tutulmuyordur.
- Sabit bir ağ kimliğine (`nginx-0`, `nginx-1` gibi sıralı hostname) ihtiyaç yoktur; Service, pod'lara round-robin şekilde yük dağıtır.
- Ölçeklendirme (scale up/down) sırasız ve anlıktır — StatefulSet'teki gibi "önce 0, sonra 1" gibi bir sıra gözetilmez.

OpenShift/Kubernetes'te bu davranışı doğal olarak sağlayan kaynak **Deployment**'tır (bir sonraki bölümde ele alacağımız **StatefulSet**'in tam tersine).

### Örnek: nginx (01 — Container Yönetimi rehberinden)

Ayrı bir örnek kurmak yerine, **[02 — Container Yönetimi](../02-container-management/README.tr.md)** rehberinde zaten oluşturduğumuz `nginx` Deployment'ını referans alıyoruz — çünkü o da klasik bir stateless web app örneğidir: 2 replikalı, `ConfigMap`'ten statik `index.html` servis eden, kendi diskinde herhangi bir veri üretmeyen/değiştirmeyen bir uygulama. (ConfigMap/Secret volume'ları salt-okunurdur ve tüm pod'larda birebir aynıdır; bu nedenle "state" sayılmazlar — uygulamanın çalışma zamanında ürettiği bir veri değildir.)

Henüz kurulu değilse:

```bash
oc apply -f ../02-container-management/namespace.yaml
oc apply -f ../02-container-management/nginx.yaml
```

### Statelessness'i kanıtlayan senaryo

**1) Mevcut pod'ları not edin:**

```bash
oc get pods -l app=nginx -o wide
```

**2) Pod'lardan birini rastgele silin** — StatefulSet'in aksine "hangi pod" silindiğinin bir önemi yoktur:

```bash
POD=$(oc get pods -l app=nginx -o jsonpath='{.items[0].metadata.name}')
oc delete pod "$POD"
```

**3) Sonucu doğrulayın** — ReplicaSet controller farklı isim/IP'li yeni bir pod oluşturur, ama:

- Service (`ClusterIP`) ve Route değişmez, istemci hiçbir şey fark etmez.
- Sayfa içeriği birebir aynıdır — çünkü veri pod'un kendi diskinde değil, ConfigMap'te tutuluyordu.
- Herhangi bir "veri kurtarma" veya "senkronizasyon" adımına gerek kalmadı.

```bash
oc get pods -l app=nginx -o wide            # yeni pod adı/IP farklı, davranış aynı
curl -s "http://$(oc get route nginx -o jsonpath='{.spec.host}')"   # içerik değişmedi
```

**4) Ölçeklendirme de sırasız çalışır** — hangi pod'un "birinci" olduğu önemsizdir (StatefulSet'teki `nginx-0`, `nginx-1` gibi sıralı kimlik yoktur, pod adları her zaman rastgele suffix'lidir):

```bash
oc scale deployment/nginx --replicas=4
oc get pods -l app=nginx -o wide
oc scale deployment/nginx --replicas=2
```

✅ **Gerçek çıktı:** Pod silinirken Route üzerinden yarım saniye arayla gönderilen **20 isteğin 20'si `200`** döndü. Silinen pod'un yerine farklı ad ve IP'li yeni bir pod geldi (`...-5rdrv/10.131.0.208` → `...-9pc7r/10.131.0.209`), sayfa içeriği değişmedi. 4 replikaya ölçeklenince rastgele ekli dört pod oluştu.

### Özet: Stateless (Deployment) vs Stateful (StatefulSet)

| | Deployment (stateless) | StatefulSet (stateful) |
|---|---|---|
| Pod adı | Rastgele suffix (`nginx-6b65f9-28w78`) | Sıralı, sabit (`postgres-0`, `postgres-1`) |
| Pod silinince | Herhangi bir sırada, veri kaybı yok | Aynı isim/kimlikle, kendi PVC'sine yeniden bağlanarak geri gelir |
| Ölçekleme sırası | Önemsiz, paralel | Sıralı (0 → 1 → 2) |
| Depolama | Genelde yok / paylaşılan salt-okunur | Her pod'un **kendi** PVC'si |

Bu karşılaştırmayı bir sonraki bölümde (**Stateful database app**) somut bir örnekle göstereceğiz.

## Stateful Database App

### Stateful nedir?

**StatefulSet**, Deployment'ın aksine her pod'a **kalıcı bir kimlik** ve **kendine ait, kalıcı depolama** (PersistentVolumeClaim) atar:

- Pod adları sıralı ve sabittir (`postgres-0`, `postgres-1`, ...) — silinip yeniden oluşsa bile **aynı isimle** geri gelir.
- Her pod, `volumeClaimTemplates` üzerinden **kendi** PVC'sine sahiptir (`data-postgres-0` gibi); pod silinse de bu PVC (ve içindeki veri) silinmez, yeni pod aynı isimle geldiğinde **aynı PVC'ye** yeniden bağlanır.
- Pod oluşturma/silme **sıralıdır** (0 → 1 → 2 sırayla oluşur, tersi sırayla silinir) — Deployment'taki gibi paralel/sırasız değildir.
- Stabil ağ kimliği için genellikle bir **headless Service** (`clusterIP: None`) kullanılır; bu sayede her pod'a `postgres-0.postgres.ocp-poc.svc.cluster.local` gibi sabit bir DNS adından ulaşılabilir.

### Örnek: PostgreSQL

`postgres.yaml` içeriği:

- **Secret (`postgres-credentials`)** — `POSTGRESQL_USER`, `POSTGRESQL_PASSWORD`, `POSTGRESQL_DATABASE`.
- **Service (`postgres`)** — `clusterIP: None` (headless), StatefulSet'in `serviceName` alanıyla eşleşir.
- **StatefulSet (`postgres`)** — `quay.io/sclorg/postgresql-16-c9s` imajı, 1 replika, `volumeClaimTemplates` ile 1Gi'lik kalıcı disk.

> **İmaj notu:** `quay.io/sclorg/postgresql-16-c9s`, Red Hat'in "Software Collections" (sclorg) projesinden gelen, **OpenShift'in rastgele UID modeliyle çalışacak şekilde tasarlanmış** bir imajdır (grup izinleri `g=u` olacak şekilde ayarlanmıştır). nginx/php-apache bölümlerindeki gibi ek bir SCC ayarına (`anyuid` vb.) **gerek duymaz** — doğru imaj seçildiğinde bu tür sorunlarla hiç karşılaşılmayabileceğinin bir örneğidir.

### `oc apply` ile oluşturma

```bash
oc project ocp-poc
oc apply -f postgres.yaml
oc get pods -l app=postgres -w
oc get pvc
```

### İmperative eşdeğer

```bash
oc create secret generic postgres-credentials \
  --from-literal=POSTGRESQL_USER=appuser \
  --from-literal=POSTGRESQL_PASSWORD=app-pass-2026 \
  --from-literal=POSTGRESQL_DATABASE=appdb

# headless service (StatefulSet'in DNS kimliği için --clusterip=None şart)
oc create service clusterip postgres --tcp=5432:5432 --clusterip=None
```

> **İmperative eşdeğer yok:** `oc create`/`kubectl create` altında Deployment, Job, CronJob için kısayollar bulunur ama **StatefulSet için imperative bir alt komut yoktur** (✅ canlı test: `oc create statefulset` → `error: Unexpected args`; yukarıdaki secret ve `clusterIP=None` headless service komutları ise sorunsuz çalıştı). StatefulSet'ler sadece `oc apply -f` (ya da `oc create -f`) ile bir YAML manifestosu üzerinden oluşturulabilir.

### Senaryo: Veri kalıcılığı ve pod kimliği

**1) Örnek tablo ve veri oluşturun:**

```bash
oc exec -it postgres-0 -- psql -U appuser -d appdb -c \
  "CREATE TABLE notes (id serial PRIMARY KEY, message text, created_at timestamptz DEFAULT now());"
oc exec -it postgres-0 -- psql -U appuser -d appdb -c \
  "INSERT INTO notes (message) VALUES ('Merhaba OpenShift POC'), ('OpenShift');"
oc exec -it postgres-0 -- psql -U appuser -d appdb -c "SELECT * FROM notes;"
```

**2) Pod'u silin ve kimliğin/verinin korunduğunu doğrulayın** — stateless senaryonun tam tersine, pod **aynı isimle** geri gelir:

```bash
oc delete pod postgres-0
oc get pods -l app=postgres -w      # yeni pod da "postgres-0" adıyla oluşur
oc exec -it postgres-0 -- psql -U appuser -d appdb -c "SELECT * FROM notes;"   # veri hâlâ orada
```

✅ **Gerçek çıktı:** Pod silindikten **19 sn** sonra yeni pod yine `postgres-0` adıyla `Ready` oldu ve aynı `data-postgres-0` PVC'sine bağlandı. Sorgu silmeden önceki iki satırı döndürdü (`1 | Merhaba OpenShift POC`, `2 | OpenShift`). Pod `restricted-v2` SCC'siyle çalıştı; ek SCC gerekmedi.

**3) Ölçeklendirmenin sıralı doğasını gözlemleyin:**

```bash
oc scale statefulset/postgres --replicas=2
oc get pods -l app=postgres -o wide     # postgres-1 oluşur, KENDİ PVC'siyle (data-postgres-1)
oc get pvc
oc scale statefulset/postgres --replicas=1
oc get pvc   # postgres-1 silinmiş olsa da data-postgres-1 PVC'si HÂLÂ DURUYOR (varsayılan davranış)
```

✅ **Gerçek çıktı:** `replicas=2` → `postgres-0`, `postgres-1` ve `data-postgres-0`, `data-postgres-1` (ikisi de `Bound`). `replicas=1` → sadece `postgres-0` kaldı, ama `data-postgres-1` PVC'si `Bound` olarak durmaya devam etti.

> **Önemli:** StatefulSet'i ölçek azaltmak ya da tamamen silmek (`oc delete -f postgres.yaml`), altındaki PVC'leri **otomatik silmez** (bu, veri kaybını önlemek için kasıtlı bir tasarımdır). Kullanılmayan PVC'leri temizlemek isterseniz elle silmeniz gerekir: `oc delete pvc -l app=postgres`.

### Nasıl silinir?

```bash
oc delete -f postgres.yaml    # StatefulSet + Service + Secret silinir, PVC'ler KALIR
oc delete pvc -l app=postgres # veriyi kalıcı olarak silmek isterseniz ek adım
```

GUI'de: **Administrator** görünümü → **Workloads > StatefulSets** → `⋮` → **Delete StatefulSet** (PVC'ler için ayrıca **Storage > PersistentVolumeClaims**'ten silme gerekir).

## API/Microservice App

### Bu senaryo neyi gösteriyor?

Bu bölümde, önceki iki bölümü birleştiren gerçekçi bir senaryo kuruyoruz: **PostgREST** — PostgreSQL şemasından otomatik olarak bir REST API üreten, ekstra kod/derleme gerektirmeyen, tamamen ortam değişkenleriyle yapılandırılan hafif bir servis. Böylece:

- **API/microservice katmanı kendisi de stateless'tir** (Deployment, 2 replika, herhangi bir pod silinebilir/scale edilebilir) — ama artık **anlamlı bir iş yapıyor**: **Stateful Database App** bölümünde oluşturduğumuz `postgres` StatefulSet'ine bağlanıp gerçek veri servis ediyor.
- Yapılandırma tamamen **ortam değişkenleri** (ConfigMap + Secret) üzerinden geliyor — dosya mount etmiyoruz, klasik bir "12-factor" mikroservis pratiği.
- Servisin kendi **health endpoint**'i (`/`) aynı zamanda veritabanı bağlantısının sağlığını da yansıtıyor — DB'ye erişemezse `/` isteği de başarısız olur, bu da readiness/liveness probe'ları için gerçekçi bir sinyal sağlar.

> Bu, "stateless" olmanın "veri işlemez" anlamına gelmediğini gösterir — sadece **veriyi kendi diskinde/belleğinde kalıcı tutmaz**; veri, ayrı ve kalıcı bir katmanda (StatefulSet + PVC) yaşar.

### Önkoşul

**Stateful Database App** bölümündeki `postgres.yaml` uygulanmış ve `notes` tablosu/verisi oluşturulmuş olmalı (bkz. yukarıdaki senaryo adım 1).

### `postgrest.yaml` içeriği

- **ConfigMap (`postgrest-config`)** — hassas olmayan ayarlar: `PGRST_DB_SCHEMA`, `PGRST_DB_ANON_ROLE`, `PGRST_SERVER_PORT`.
- **Secret (`postgrest-db-uri`)** — bağlantı dizesi (kullanıcı adı/şifre içerdiği için Secret'ta): `PGRST_DB_URI`.
- **Deployment (`postgrest`)** — 2 replikalı, `postgrest/postgrest` imajı, port 3000.
- **Service** ve **Route (`postgrest`)** — dışarıdan erişim için.

> Bu POC'de basitlik için `PGRST_DB_ANON_ROLE` olarak doğrudan bağlantı sahibi `appuser` kullanılmıştır. Üretimde bunun yerine sadece gerekli tablolara `SELECT` izni olan, ayrı ve kısıtlı bir salt-okunur rol tanımlanması önerilir.

### `oc apply` ile oluşturma

```bash
oc apply -f postgrest.yaml
oc get pods -l app=postgrest -w
```

### Test: API üzerinden veriyi okuma

```bash
HOST=$(oc get route postgrest -o jsonpath='{.spec.host}')
curl -s "http://$HOST/"          # kök endpoint: OpenAPI şeması (200 dönüyorsa DB bağlantısı sağlıklı demektir)
curl -s "http://$HOST/notes"     # postgres StatefulSet'inden gelen gerçek veri, JSON olarak
```

✅ **Gerçek çıktı:** `GET /` → `200`. `GET /notes` → `[{"id":1,"message":"Merhaba OpenShift POC",...}, {"id":2,"message":"OpenShift",...}]`.

### Senaryo: API katmanının stateless dayanıklılığı

```bash
POD=$(oc get pods -l app=postgrest -o jsonpath='{.items[0].metadata.name}')
oc delete pod "$POD"
curl -s "http://$HOST/notes"     # Service diğer replikaya yönlendirdiği için kesinti hissedilmez
oc scale deployment/postgrest --replicas=4
oc scale deployment/postgrest --replicas=2
```

✅ **Gerçek çıktı:** PostgREST pod'u silinirken gönderilen **20 isteğin 20'si `200`** döndü; 4 replikaya ölçekleme sorunsuz tamamlandı.

Bu davranış **Stateless Web App** bölümündeki nginx senaryosuyla birebir aynıdır — fark, bu kez API'nin arkasında gerçek ve kalıcı bir veri kaynağı (StatefulSet) olmasıdır.

### Nasıl silinir?

```bash
oc delete -f postgrest.yaml
```

GUI'de: **Developer** görünümünde **Topology**'den `postgrest` Deployment'ına tıklayıp **Actions > Delete Deployment**, ya da **Administrator** görünümünde **Workloads > Deployments** üzerinden silinebilir.
