# 16 — OADP (Velero) Backup/Restore Demo Rehberi — ODF S3 Üzerinde

> 🇬🇧 [English](README.md) · [← 15 — ODF Performans Testi](../15-odf-performance-test/README.tr.md) · [POC akışı](../../README.tr.md) · [17 — OpenShift Virtualization →](../../E-virtualization/17-virtualization/README.tr.md)

Bu doküman, **OADP (OpenShift API for Data Protection / Velero)** ile bir namespace'in tamamen silinip **backup'tan geri yüklenmesini** anlatır. Depolama için ([12 — Logging](../12-logging/README.tr.md) rehberindeki gibi) cluster'da zaten kurulu olan **ODF (Ceph RGW, S3-uyumlu)** kullanılmıştır. Tüm adımlar lab ortamında (OpenShift 4.22, OADP 1.6) **uçtan uca canlı test edilmiştir**: `poc-oadp-demo` namespace'i içeriğiyle (Deployment, ConfigMap, Secret, Service **ve veri yazılmış bir PVC**) yedeklenmiş, namespace tamamen silinmiş, sonra backup'tan tam olarak geri yüklenmiş ve PVC'deki dosyaların MD5 özetleri birebir aynı çıkmıştır.

Senaryo sırası:

1. Ön koşul kontrolü (OADP operatörü, ODF S3 backend)
2. S3 bucket + credentials hazırlama (ODF/Ceph RGW)
3. `DataProtectionApplication` (DPA) oluşturma
4. Test namespace'i ve içinde kaynaklar oluşturma
5. Backup alma
6. Namespace'i silme (felaket simülasyonu)
7. Restore etme ve doğrulama (gerçek test çıktıları)
8. Bilinen uyarılar (zararsız, canlı testte görülen)
9. Temizlik

---

## 1. Ön Koşul Kontrolü

**a) OADP operatörü kurulu mu?**

```bash
oc get csv -n openshift-adp | grep -i oadp
```

Lab ortamında OADP operatörü (`oadp-operator.v1.6.1`) zaten `openshift-adp` namespace'inde kuruluydu.

Kurulu değilse:

```bash
cat <<'EOF' | oc apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-adp
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-adp
  namespace: openshift-adp
spec:
  targetNamespaces:
    - openshift-adp
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: redhat-oadp-operator
  namespace: openshift-adp
spec:
  channel: stable-1.6
  name: redhat-oadp-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  installPlanApproval: Automatic
EOF
```

**b) S3 backend için ODF sağlıklı mı?**

[12 — Logging](../12-logging/README.tr.md) rehberindeki gibi bu rehberde de S3 backend olarak **Ceph RGW** (`ocs-storagecluster-ceph-rgw`) kullanılır (lab ortamında NooBaa bir dönem sağlıksızdı). Kendi ortamınızda önce ikisini de kontrol edin:

```bash
oc get noobaa -n openshift-storage
oc get cephobjectstore -n openshift-storage
```

---

## 2. S3 Bucket + Credentials (ODF/Ceph RGW)

```yaml
# oadp-bucket-obc.yaml
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  name: oadp-bucket-odf
  namespace: openshift-adp
spec:
  generateBucketName: oadp-bucket-odf
  storageClassName: ocs-storagecluster-ceph-rgw
```

```bash
oc apply -f oadp-bucket-obc.yaml
oc get obc oadp-bucket-odf -n openshift-adp
```

✅ **Gerçek çıktı:** `PHASE: Bound` (birkaç saniye içinde).

Velero'nun beklediği **AWS credentials INI formatında** bir secret üretin:

```bash
ACCESS_KEY=$(oc get secret oadp-bucket-odf -n openshift-adp -o jsonpath='{.data.AWS_ACCESS_KEY_ID}' | base64 -d)
SECRET_KEY=$(oc get secret oadp-bucket-odf -n openshift-adp -o jsonpath='{.data.AWS_SECRET_ACCESS_KEY}' | base64 -d)

cat > cloud-credentials <<EOF
[default]
aws_access_key_id=${ACCESS_KEY}
aws_secret_access_key=${SECRET_KEY}
EOF

oc create secret generic cloud-credentials -n openshift-adp --from-file=cloud=cloud-credentials
```

> **Not:** Anahtar dosyanın içindeki key adı **`cloud`** olmalı — DPA'nın `credential.key` alanı bunu referans alır.

---

## 3. DataProtectionApplication (DPA)

```yaml
# dpa.yaml
apiVersion: oadp.openshift.io/v1alpha1
kind: DataProtectionApplication
metadata:
  name: dpa-odf
  namespace: openshift-adp
spec:
  configuration:
    velero:
      defaultPlugins:
        - openshift
        - aws
        - csi        # PVC'leri CSI snapshot ile yedeklemek için (Bölüm 5.1)
        - kubevirt   # VM yedeklemek için (17 — Virtualization); VM yoksa gerekmez
    nodeAgent:
      enable: false          # PV dosya-seviyesi (kopia/restic) yedeklemesi kapalı — bkz. Bölüm 8
      uploaderType: kopia     # enable:false olsa da alan zorunlu (CRD validasyonu)
  backupLocations:
    - velero:
        provider: aws
        default: true
        credential:
          name: cloud-credentials
          key: cloud
        objectStorage:
          bucket: REPLACE_ME_BUCKET_NAME   # Bolum 2'deki OBC'nin urettigi gercek bucket adi
          prefix: velero
        config:
          region: us-east-1
          s3Url: http://rook-ceph-rgw-ocs-storagecluster-cephobjectstore.openshift-storage.svc:80
          s3ForcePathStyle: "true"
```

```bash
BUCKET=$(oc get cm oadp-bucket-odf -n openshift-adp -o jsonpath='{.data.BUCKET_NAME}')
sed "s/REPLACE_ME_BUCKET_NAME/${BUCKET}/" dpa.yaml | oc apply -f -

oc get backupstoragelocation -n openshift-adp
oc get pods -n openshift-adp
```

✅ **Gerçek çıktı:** `backupstoragelocation dpa-odf-1` → `PHASE: Available`, `DEFAULT: true`; `velero-xxxxx` pod'u `1/1 Running`.

> **Not ([12 — Logging](../12-logging/README.tr.md) rehberindeki gibi):** RGW endpoint'i **HTTP (port 80)** ile verildi, HTTPS değil — trafik cluster-internal (ClusterIP) olduğu için POC'de sertifika/CA karmaşasından kaçınmak amacıyla. Üretimde CA bundle ile HTTPS tercih edin.

---

## 4. Test Namespace'i ve Kaynaklar

```yaml
# demo-workload.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: poc-oadp-demo
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: demo-config
  namespace: poc-oadp-demo
data:
  greeting: "hello-oadp-restore-test"
  created-at: "2026-08-12"
---
apiVersion: v1
kind: Secret
metadata:
  name: demo-secret
  namespace: poc-oadp-demo
type: Opaque
stringData:
  password: "s3cr3t-poc-oadp-demo"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
  namespace: poc-oadp-demo
  labels:
    app: demo-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo-app
  template:
    metadata:
      labels:
        app: demo-app
    spec:
      containers:
        - name: demo
          image: busybox:1.36
          command: ["sh", "-c", "echo $(GREETING) from $(POD_NAME); sleep 3600"]
          env:
            - name: GREETING
              valueFrom:
                configMapKeyRef:
                  name: demo-config
                  key: greeting
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
---
apiVersion: v1
kind: Service
metadata:
  name: demo-app
  namespace: poc-oadp-demo
spec:
  selector:
    app: demo-app
  ports:
    - port: 8080
      targetPort: 8080
```

```bash
oc apply -f demo-workload.yaml
oc get deployment,cm,secret,svc -n poc-oadp-demo
```

✅ **Gerçek çıktı:** `deployment.apps/demo-app` → `2/2 READY`.

---

## 5. Backup Alma

```yaml
# backup.yaml
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: poc-oadp-demo-backup
  namespace: openshift-adp
spec:
  includedNamespaces:
    - poc-oadp-demo
  storageLocation: dpa-odf-1
  ttl: 720h0m0s
```

```bash
oc apply -f backup.yaml
oc get backup.velero.io poc-oadp-demo-backup -n openshift-adp -o jsonpath='{.status}'
```

> **Dikkat:** `oc get backup` (tam kaynak adı vermeden) bu cluster'da **`backups.postgresql.cnpg.noobaa.io`** (CloudNativePG operatörünün kaynak tipi) ile çakışıp yanlış kaynağı sorgulayabilir — kısa isim çakışması. Her zaman **`oc get backup.velero.io`** kullanın.

✅ **Gerçek çıktı:**

```json
{"completionTimestamp":"...","phase":"Completed","progress":{"itemsBackedUp":58,"totalItems":58},...}
```

### 5.1 PVC verisiyle birlikte yedekleme (CSI snapshot)

Kalıcı diski olan uygulamalar için `demo-pvc.yaml` (1Gi PVC + ona yazan `data-app`) eklenir ve PVC'ye veri yazılır:

```bash
oc apply -f demo-pvc.yaml
P=$(oc get pod -n poc-oadp-demo -l app=data-app -o jsonpath='{.items[0].metadata.name}')
oc exec $P -n poc-oadp-demo -- /bin/sh -c 'echo "kalici-veri $(date -u +%FT%TZ)" > /data/kritik.txt'
oc exec $P -n poc-oadp-demo -- dd if=/dev/urandom of=/data/rastgele.bin bs=1M count=20
oc exec $P -n poc-oadp-demo -- md5sum /data/kritik.txt /data/rastgele.bin
```

Yedek, `backup.yaml` yerine `backup-with-volumes.yaml` ile alınır: `snapshotVolumes: true` ve hangi VolumeSnapshotClass'ın kullanılacağını belirten bir annotation içerir (DPA'da `csi` plugin'i gerekir).

```bash
oc apply -f backup-with-volumes.yaml
oc get backups.velero.io poc-oadp-demo-backup -n openshift-adp \
  -o jsonpath='{.status.phase} {.status.progress.itemsBackedUp}/{.status.progress.totalItems} csi={.status.csiVolumeSnapshotsCompleted}/{.status.csiVolumeSnapshotsAttempted}'
```

✅ **Gerçek çıktı:** **22 sn**'de `Completed 75/75 csi=1/1`, hata/uyarı yok.

> Velero normalde `velero.io/csi-volumesnapshot-class=true` etiketli VolumeSnapshotClass'ı arar. Etiket yoksa ve paylaşımlı VolumeSnapshotClass'ı değiştirmek istemiyorsanız, annotation yöntemi (`velero.io/csi-volumesnapshot-class_<driver>: <class>`) kullanılabilir.

> **Kapsam:** CSI snapshot'lar Ceph içinde tutulur, S3'e sadece Kubernetes nesneleri yazılır. Aynı cluster'a geri dönüş için yeterlidir; Ceph'in kaybı ya da başka bir cluster'a taşıma için DPA'da `nodeAgent.enable: true` ve Backup'ta `snapshotMoveData: true` (Data Mover) gerekir.

---

## 6. Namespace'i Silme (Felaket Simülasyonu)

```bash
oc delete namespace poc-oadp-demo --wait=true
oc get ns poc-oadp-demo
```

✅ **Gerçek çıktı:** `Error from server (NotFound): namespaces "poc-oadp-demo" not found` — namespace, içindeki Deployment/ConfigMap/Secret/Service ile birlikte **tamamen silindi**.

---

## 7. Restore ve Doğrulama (gerçek test çıktıları)

```yaml
# restore.yaml
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: poc-oadp-demo-restore
  namespace: openshift-adp
spec:
  backupName: poc-oadp-demo-backup
```

```bash
oc apply -f restore.yaml
oc get restore.velero.io poc-oadp-demo-restore -n openshift-adp -o jsonpath='{.status}'
```

✅ **Gerçek çıktı:**

```json
{"completionTimestamp":"...","phase":"Completed","progress":{"itemsRestored":44,"totalItems":44},"warnings":10}
```

(PVC'li ikinci test turunda: **38 sn**'de `Completed`, `49/49`, 12 uyarı.)

**Kaynakların ve verilerin gerçekten geri geldiğini doğrulama:**

```bash
oc get deployment,cm,secret,svc -n poc-oadp-demo
oc get cm demo-config -n poc-oadp-demo -o jsonpath='{.data}'
oc get secret demo-secret -n poc-oadp-demo -o jsonpath='{.data.password}' | base64 -d
```

✅ **Gerçek çıktı:**

```
deployment.apps/demo-app   2/2 READY   (yeniden hiç müdahale etmeden Running'e geçti)

{"created-at":"2026-08-12","greeting":"hello-oadp-restore-test"}   <-- ConfigMap verisi birebir geri geldi

s3cr3t-poc-oadp-demo   <-- Secret icerigi birebir geri geldi
```

**PVC verisinin doğrulanması:**

```bash
P=$(oc get pod -n poc-oadp-demo -l app=data-app -o jsonpath='{.items[0].metadata.name}')
oc exec $P -n poc-oadp-demo -- md5sum /data/kritik.txt /data/rastgele.bin
```

✅ **Gerçek çıktı:** Yedek öncesi ve restore sonrası MD5'ler birebir aynı (`71a1aad6...  /data/kritik.txt`, `f3910388...  /data/rastgele.bin`); `kritik.txt` içeriği `kalici-veri 2026-10-05T23:07:57Z`. PVC, CSI snapshot'tan yeniden oluşturuldu ve `data-app` kendiliğinden ona bağlanıp açıldı.

**Sonuç: namespace, Deployment (replikalarıyla), ConfigMap, Secret ve PVC verisi — hepsi içerikleriyle birebir restore edildi.**

---

## 8. Bilinen Uyarılar (canlı testte görülen, zararsız)

Restore `Completed` oldu ama **10 uyarı** verdi. `velero restore describe --details` ile incelendi:

```bash
POD=$(oc get pods -n openshift-adp -l app.kubernetes.io/name=velero -o jsonpath='{.items[0].metadata.name}')
oc exec -n openshift-adp "$POD" -c velero -- ./velero restore describe poc-oadp-demo-restore --details
```

Tüm uyarılar şu iki kategoriden:

1. **Cluster-scoped kaynaklar zaten mevcut** (`CustomResourceDefinition:clusterserviceversions.operators.coreos.com`, `SecurityContextConstraints:restricted-v2`) — bunlar cluster genelinde paylaşılan kaynaklar, backup'taki haliyle üzerine yazılmadı (**doğru davranış**, aksi halde cluster'ın geri kalanını etkilerdi).
2. **Namespace içinde otomatik üretilen kaynaklar zaten mevcut** (`pipeline-dockercfg-xxxxx` secret, `istio-ca-crl`/`istio-ca-root-cert`/`kube-root-ca.crt`/`openshift-service-ca.crt` ConfigMap'leri, `openshift-pipelines-edit`/`pipelines-scc-rolebinding` RoleBinding'leri) — namespace yeniden oluşturulur oluşturulmaz OpenShift'in kendi controller'ları (service account controller, CA injector, OpenShift Pipelines operatörü vb.) bunları **zaten otomatik olarak yeniden üretiyor**; Velero restore çalıştığında bunlar zaten mevcut olduğu için "already exists" uyarısı veriyor. **Gerçek bir veri kaybı değil.**

**Pratik sonuç:** Bu tür uyarıları görmek normal ve beklenen — asıl kontrol edilmesi gereken, **sizin uygulamanıza ait** kaynakların (Deployment, ConfigMap, Secret, kendi RoleBinding'leriniz vb.) içerikleriyle geri gelip gelmediğidir (Bölüm 7'de doğrulandı).

---

## 9. Temizlik

> ⚠️ **Restore'un bıraktığı snapshot:** PVC restore edilirken Velero, namespace'te `deletionPolicy: Retain` olan bir VolumeSnapshot/VolumeSnapshotContent oluşturur. Namespace silinse bile bu snapshot **Ceph'te kalır** (canlı testte doğrulandı). Silmeden önce politikası `Delete` yapılmalıdır.
>
> Ayrıca `oc delete backup.velero.io` sadece Kubernetes nesnesini siler; S3'teki yedek verisi ve CSI snapshot'lar için **`DeleteBackupRequest`** kullanılmalıdır (Velero restore kayıtlarını da kendisi temizler).

```bash
# Restore'un bıraktığı Retain snapshot'ları Delete'e çevir
for c in $(oc get volumesnapshot -n poc-oadp-demo -o jsonpath='{.items[*].status.boundVolumeSnapshotContentName}'); do
  oc patch volumesnapshotcontent $c --type=merge -p '{"spec":{"deletionPolicy":"Delete"}}'
done
# Backup'ı S3 verisi ve snapshot'larıyla birlikte sil
cat <<'EOF' | oc create -f -
apiVersion: velero.io/v1
kind: DeleteBackupRequest
metadata:
  generateName: poc-oadp-demo-backup-delete-
  namespace: openshift-adp
spec:
  backupName: poc-oadp-demo-backup
EOF
oc delete ns poc-oadp-demo
# DPA'yı başka rehberler (17 — Virtualization) kullanmayacaksa:
oc delete dpa dpa-odf -n openshift-adp
oc delete secret cloud-credentials -n openshift-adp
oc delete obc oadp-bucket-odf -n openshift-adp
```

---

## Özet Tablo

| Bileşen | Namespace | Rolü |
|---|---|---|
| OADP Operator | `openshift-adp` | Velero'yu ve `DataProtectionApplication` CRD'sini yönetir |
| `ObjectBucketClaim` (Ceph RGW) | `openshift-adp` | Backup'ların yazılacağı S3 bucket'ı sağlar |
| `DataProtectionApplication` (`dpa-odf`) | `openshift-adp` | Velero'nun hangi S3'e, hangi credential'la yazacağını tanımlar |
| `Backup` (`velero.io`) | `openshift-adp` | Belirtilen namespace'in tüm Kubernetes kaynaklarını S3'e yedekler |
| `Restore` (`velero.io`) | `openshift-adp` | Bir backup'tan namespace + kaynakları geri yükler |

**Altın kurallar:**
1. `oc get backup` yerine her zaman **`oc get backup.velero.io`** kullanın — bu cluster'da CloudNativePG'nin `backups` kaynağıyla isim çakışması var.
2. NooBaa sağlıksızsa (bkz. [12 — Logging README](../12-logging/README.tr.md)) **Ceph RGW**'ye geçin — OADP için de aynı S3 backend mantığı geçerli.
3. Restore'daki "already exists" uyarıları çoğunlukla **zararsızdır** — asıl doğrulama, kendi uygulama kaynaklarınızın **içerikleriyle** (ConfigMap/Secret verisi, replika sayısı vb.) geri geldiğini kontrol etmektir, sadece `phase: Completed`'e güvenmeyin.
4. PVC verisi **CSI snapshot** ile yedeklendi ve doğrulandı (Bölüm 5.1). Dosya seviyesi yedekleme (`nodeAgent`/kopia, `defaultVolumesToFsBackup: true`) ya da verinin S3'e kopyalanması (`snapshotMoveData`) ayrıca açılabilir; bu POC'de kapalıdır.
5. Restore'un bıraktığı `Retain` snapshot'ları ve S3'teki yedekleri temizlerken `DeleteBackupRequest` kullanın (Bölüm 9).
