# 14 — OpenShift'te ODF (S3) ile MultiClusterObservability Kurulumu — Uçtan Uca CLI Rehberi

> 🇬🇧 [English](README.md) · [← 13 — East-West Trafik Kontrolü](../13-east-west-traffic-control/README.tr.md) · [POC akışı](../../README.tr.md) · [15 — ODF Performans Testi →](../15-odf-performance-test/README.tr.md)

Bu doküman, Red Hat Advanced Cluster Management (ACM) bileşeni olan
**MultiClusterObservability (MCO)**'yu, metrik depolama (Thanos object storage)
için **OpenShift Data Foundation (ODF)**'in S3 uyumlu **NooBaa** Multi-Cloud
Gateway'i üzerinden oluşturulan bir bucket kullanarak, tamamen `oc` CLI ile
kurma adımlarını anlatır.

> Tüm adımlar lab ortamında (OpenShift 4.22, ACM hub + 2 managed cluster)
> **uçtan uca canlı test edilmiştir**; "✅ Gerçek çıktı" satırları bu testten alınmıştır.

## Ön koşullar

- `oc` ile küme-yönetici (cluster-admin) yetkisiyle giriş yapılmış olmalı.
- **ACM / MultiClusterHub** kurulu ve `Running` durumda olmalı:

  ```bash
  oc get multiclusterhub -A
  ```

- **ODF** operatörü kurulu ve bir `StorageCluster` mevcut olmalı:

  ```bash
  oc get csv -n openshift-storage | grep odf-operator
  oc get storagecluster -n openshift-storage
  ```

- MCO CRD'lerinin küme üzerinde var olduğunu doğrulayın (ACM observability
  bileşeni ile birlikte gelir):

  ```bash
  oc get crd multiclusterobservabilities.observability.open-cluster-management.io
  ```

## 1. Adım — ODF/NooBaa üzerinden S3 bucket oluşturma (ObjectBucketClaim)

ODF, S3 bucket taleplerini Kubernetes native bir kaynak olan
`ObjectBucketClaim` (OBC) ile karşılar. `openshift-storage.noobaa.io`
storage class'ı NooBaa'nın kendi S3 servisini kullanarak bucket'ı otomatik
oluşturur.

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

OBC'nin `Bound` olmasını bekleyin:

```bash
oc wait --for=jsonpath='{.status.phase}'=Bound \
  obc/multiclusterobs -n openshift-storage --timeout=120s
```

OBC bound olduğunda ODF otomatik olarak aynı isimde bir **Secret** (erişim
anahtarları) ve bir **ConfigMap** (bucket/endpoint bilgisi) üretir:

```bash
oc get obc multiclusterobs -n openshift-storage
oc get secret multiclusterobs -n openshift-storage
oc get cm multiclusterobs -n openshift-storage -o yaml
```

ConfigMap içeriği örnek:

```yaml
data:
  BUCKET_HOST: s3.openshift-storage.svc
  BUCKET_NAME: multiclusterobs-<uuid>
  BUCKET_PORT: "443"
```

## 2. Adım — Bucket bilgilerini ve kimlik bilgilerini okuma

```bash
BUCKET_HOST=$(oc get cm multiclusterobs -n openshift-storage -o jsonpath='{.data.BUCKET_HOST}')
BUCKET_NAME=$(oc get cm multiclusterobs -n openshift-storage -o jsonpath='{.data.BUCKET_NAME}')
ACCESS_KEY=$(oc get secret multiclusterobs -n openshift-storage -o jsonpath='{.data.AWS_ACCESS_KEY_ID}' | base64 -d)
SECRET_KEY=$(oc get secret multiclusterobs -n openshift-storage -o jsonpath='{.data.AWS_SECRET_ACCESS_KEY}' | base64 -d)
```

Küme içi NooBaa S3 servisinin adresini de doğrulayın (LoadBalancer/ClusterIP
servis, port 80/443):

```bash
oc get svc s3 -n openshift-storage
```

## 3. Adım — Observability namespace'i ve Thanos object-storage secret'ı

MCO'nun bileşenleri `open-cluster-management-observability` namespace'inde
çalışır; bu namespace ACM/MCH kurulumuyla zaten oluşmuş olmalıdır (yoksa
`oc create ns open-cluster-management-observability` ile oluşturulabilir).

Thanos'un S3'e nasıl bağlanacağını tanımlayan `thanos.yaml` içerikli bir
Secret oluşturun:

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

# Secret zaten varsa (önceki bir kurulumdan) "create" hata verir; aşağıdaki satır yoksa oluşturur, varsa günceller:
oc create secret generic thanos-object-storage \
  -n open-cluster-management-observability \
  --from-file=thanos.yaml=/tmp/thanos.yaml --dry-run=client -o yaml | oc apply -f -

rm -f /tmp/thanos.yaml
```

> ⚠️ **Önceden kalmış secret:** `thanos-object-storage` daha önceki bir MCO kurulumundan kalmış olabilir; artık var olmayan bir bucket'ı gösteriyorsa Thanos veri yazamaz. Lab testinde aylar öncesinden kalma, OBC'si silinmiş bir secret bulundu; önce yedeklenip yeni bucket bilgileriyle güncellendi. Kontrol: `oc get secret thanos-object-storage -n open-cluster-management-observability -o jsonpath='{.metadata.creationTimestamp}'` ve `oc get obc -A`.

> **Not:** `endpoint` alanında port **80** ve `insecure: true` kullanılıyor
> çünkü küme içinden NooBaa S3 servisine düz HTTP ile erişiliyor (trafik
> cluster network'ünden çıkmıyor). Kümeler arası / dışarıdan erişimde TLS
> (443, `insecure: false`) tercih edilmelidir.

## 4. Adım — Kullanılacak StorageClass'ı belirleme

MCO'nun Thanos bileşenleri (Alertmanager, Compact, Receive, Rule, Store)
kendi PVC'leri için bir blok depolama StorageClass'ına ihtiyaç duyar. ODF'in
varsayılan RBD class'ı genelde uygundur:

```bash
oc get storageclass
# örn: ocs-storagecluster-ceph-rbd (default)
```

## 5. Adım — MultiClusterObservability CR'ını oluşturma

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

Alan notları:

| Alan | Açıklama |
|---|---|
| `metricObjectStorage.name/key` | 3. adımda oluşturulan Secret'ın adı ve içindeki key |
| `storageClass` | PVC'ler için kullanılacak blok depolama sınıfı |
| `*StorageSize` | Her Thanos bileşeninin PVC boyutu (ihtiyaca göre ayarlanır) |

`v1beta2` API'sinde `observabilityAddonSpec` ve `storageConfig` zorunlu
alanlardır; şema doğrulaması CRD üzerinden kontrol edilebilir:

```bash
oc get crd multiclusterobservabilities.observability.open-cluster-management.io -o json \
  | jq '.spec.versions[] | select(.storage==true) | .schema.openAPIV3Schema.properties.spec.required'
```

## 6. Adım — Kurulumu doğrulama

```bash
# CR durumu
oc get multiclusterobservability observability -o jsonpath='{range .status.conditions[*]}{.type}={.status} ({.reason}): {.message}{"\n"}{end}'

# Pod'ların ayağa kalkışı
oc get pods -n open-cluster-management-observability -w
```

Kurulum tamamlandığında `Installing=False`, `Ready=True` koşulları
görünmelidir ve namespace'te şu bileşenler `Running`/`Ready` olmalıdır:

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

Grafana'ya erişim için route:

```bash
oc get route -n open-cluster-management-observability
```

✅ **Gerçek çıktı:** MCO `Ready=True` **123 sn**'de. Birkaç dakika içinde yukarıdaki bileşenlerin hepsi `Running` oldu (thanos-receive ×3, store-shard ×3, query/query-frontend, observatorium-api ×2, grafana ×2, alertmanager ×3, metrics-collector). Route'lar: `alertmanager`, `grafana`, `observatorium-api`, `rbac-query-proxy`.

**Managed cluster'lardaki addon:**

```bash
oc get managedclusteraddon observability-controller -A
```

✅ **Gerçek çıktı:** Erişilebilir managed cluster'da `Available=True (ManagedClusterAddOnLeaseUpdated)`. O sırada zaten erişilemeyen (`ManagedCluster available=Unknown`) cluster'da `Unknown`. Hub cluster için ayrı addon yoktur; hub'ın metrikleri `metrics-collector-deployment` ile toplanır.

**Metriklerin gerçekten aktığını doğrulama** (pod'ların `Running` olması yeterli değildir):

```bash
oc port-forward -n open-cluster-management-observability svc/observability-thanos-query-frontend 19090:9090 &
curl -s --get --data-urlencode 'query=count by (cluster) (up)' http://127.0.0.1:19090/api/v1/query
```

✅ **Gerçek çıktı (kurulumdan ~5 dk sonra):**

| cluster | `count(up)` | `count({__name__=~"cluster:.*"})` |
|---|---|---|
| hub | 357 | 523 |
| managed cluster | 130 | 516 |

Kurulumdan hemen sonra collector logunda bir kez `unable to forward results ... observatorium-api` hatası görüldü (API henüz hazır değildi); sonraki gönderimler `metrics pushed successfully` oldu.

## Bilinen davranışlar / sık karşılaşılan durumlar

- **`observability-thanos-store-shard-*` pod'ları uzun süre `0/1 Running`
  kalabilir.** Bucket'ta önceden yüklenmiş çok sayıda Thanos bloğu varsa
  (ör. daha önce MCO kurulup silinmiş ama bucket verisi silinmemişse), store
  gateway her bloğun index header'ını tek tek yükler. Bu süreç normaldir,
  hata değildir; ilerlemeyi şu şekilde izleyebilirsiniz:

  ```bash
  oc logs observability-thanos-store-shard-0-0 \
    -n open-cluster-management-observability -f | grep "loaded new block"
  ```

  Yavaşsa ama `thanos_blocks_meta_sync_failures_total` metriği `0` ise
  (pod içinden `curl localhost:10902/metrics` ile kontrol edilebilir),
  gerçek bir sorun yoktur, sadece zaman alıyordur.

- **`FailedToRetrieveImagePullSecret: multiclusterhub-operator-pull-secret`**
  uyarısı events'te görülebilir. İmajlar node üzerinde zaten cache'liyse
  (`Pulled ... already present on machine`) bu engelleyici değildir; yeni bir
  node'a scheduling olduğunda image pull hatası verirse, ACM/MCH'nin global
  pull secret'ının ilgili namespace'e senkronize olduğundan emin olun.

- **Bucket'ı yeniden kullanmak isterseniz**: Eğer daha önce oluşturulmuş bir
  OBC/Secret zaten `Bound` durumdaysa (`oc get obc -A`), yeni bir bucket
  oluşturmak yerine doğrudan 3. adıma (mevcut bucket bilgileriyle
  `thanos-object-storage` secret'ını oluşturma) geçebilirsiniz.

## Kaynakların temizlenmesi (isteğe bağlı, geri alınamaz)

```bash
oc delete multiclusterobservability observability
oc delete pvc --all -n open-cluster-management-observability   # MCO silinince 13 PVC (~250Gi) geride kalır
oc delete secret thanos-object-storage -n open-cluster-management-observability
oc delete obc multiclusterobs -n openshift-storage   # bucket verisini de siler
```

✅ **Gerçek çıktı:** MCO CR'ı silinince pod'lar ve managed cluster'lardaki `observability-controller` addon'ları otomatik kaldırıldı, ama Thanos/Alertmanager'ın **13 PVC'si kaldı**; elle silinmesi gerekti.

> `obc` silmek NooBaa bucket'ındaki **tüm veriyi kalıcı olarak siler**.
> Sadece MCO CR'ını silmek bucket'ı ve içindeki metrik verisini korur.
