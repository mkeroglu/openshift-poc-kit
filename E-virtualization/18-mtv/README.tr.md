# 18 — MTV ile VMware / OVA'dan Taşıma

> 🇬🇧 [English](README.md) · [← 17.1 — Windows Boot Source](../17-virtualization/windows/README.tr.md) · [POC akışı](../../README.tr.md) · [19 — NodeHealthCheck →](../19-node-health-check/README.tr.md)

Bu rehber, **Migration Toolkit for Virtualization (MTV)** ile mevcut sanallaştırma platformundaki (VMware vSphere, OVA, RHV, OpenStack) VM'lerin OpenShift Virtualization'a taşınmasını anlatır: provider tanımı, envanter, ağ/depolama haritaları, plan ve taşıma.

Testler lab ortamında (OpenShift 4.22, OpenShift Virtualization 4.22.9, **MTV 2.12.9**) **OVA provider** ile yapıldı; canlı vCenter VM'lerine dokunulmadı. Plan ve haritalara kadar her adım çalıştı, ama **bu MTV sürümünde OVA taşıması disk aşamasında başarısız oldu**. Kök neden aşağıda kanıtlarıyla anlatılıyor (bölüm 7). vSphere akışı aynı kaynak tipleriyle çalışır; farkları bölüm 2 ve 8'de.

Senaryo sırası:

1. Kavramlar ve ön koşullar
2. Provider tanımı (vSphere ve OVA)
3. Envanter: taşınacak VM, ağ ve disk ID'leri
4. NetworkMap ve StorageMap
5. Plan
6. Migration (taşımayı başlatma) ve izleme
7. Canlı testte görülenler (OVA + MTV 2.12.9)
8. vSphere'e özgü notlar (warm migration, VDDK)
9. Temizlik

---

## 1. Kavramlar ve Ön Koşullar

| Kaynak | Rolü |
|---|---|
| `Provider` | Kaynak (vSphere/OVA/RHV/OpenStack) ve hedef (`host` = bu cluster) platform bağlantısı |
| `NetworkMap` | Kaynaktaki ağların hedefteki karşılığı (pod ağı ya da Multus NAD) |
| `StorageMap` | Kaynaktaki datastore'ların / disklerin hedefteki storage class'ı |
| `Plan` | Hangi VM'ler, hangi haritalarla, hangi namespace'e taşınacak |
| `Migration` | Planın bir çalıştırılması |

Taşıma sırasında MTV, VM diskini hedef PVC'ye aktarır ve **virt-v2v** ile misafir işletim sistemini KVM'e uyarlar (virtio sürücüleri, guest agent, boot ayarları). Ardından aynı CPU/bellek/firmware ile bir `VirtualMachine` oluşturur.

```bash
oc get csv -n openshift-mtv | grep mtv-operator             # MTV operatörü
oc get forkliftcontroller -n openshift-mtv                   # yoksa: Installed Operators -> MTV -> ForkliftController oluştur
oc get providers.forklift.konveyor.io -n openshift-mtv       # "host" (hedef) provider'ı otomatik gelir
```

> Kısa ad çakışması: ACM gibi operatörler kuruluysa `oc get plan` / `oc get provider` başka kaynakları getirebilir. Her zaman tam adı kullanın: `plans.forklift.konveyor.io`, `providers.forklift.konveyor.io`.

✅ **Gerçek çıktı:** `mtv-operator.v2.12.9 Succeeded`; provider'lar: `host` (openshift, `Ready`), bir vSphere provider (`Ready`), `ova` (`Ready`).

**Console:** **Migration** menüsü (MTV kurulunca sol menüde açılır) → **Providers / Plans / NetworkMaps / StorageMaps**.

---

## 2. Provider Tanımı

### 2.1 vSphere

```bash
oc create secret generic vsphere-credentials -n openshift-mtv \
  --from-literal=user='<vcenter-kullanicisi>' --from-literal=password='<sifre>' \
  --from-literal=url='https://<vcenter>/sdk' --from-literal=insecureSkipVerify=true
```

```yaml
apiVersion: forklift.konveyor.io/v1beta1
kind: Provider
metadata:
  name: vcenter
  namespace: openshift-mtv
spec:
  type: vsphere
  url: https://<vcenter>/sdk
  secret: {name: vsphere-credentials, namespace: openshift-mtv}
  settings:
    vddkInitImage: <registry>/vddk:<sürüm>   # önerilir: VMware VDDK kütüphanesiyle oluşturulmuş init imajı
    sdkEndpoint: vcenter                      # ya da esxi
```

- Kullanıcı en az salt-okuma + VM snapshot/disk okuma yetkisine sahip olmalıdır (MTV dokümanındaki "vSphere privileges" listesi).
- **VDDK imajı** vSphere'den disk aktarımını belirgin şekilde hızlandırır; VMware'in VDDK paketinden bir kez oluşturulup iç registry'ye konur.

### 2.2 OVA (NFS paylaşımı)

OVA provider, `.ova` dosyalarının (ya da açılmış `.ovf` + disk dizinlerinin) bulunduğu bir **NFS paylaşımını** tarar:

```yaml
apiVersion: forklift.konveyor.io/v1beta1
kind: Provider
metadata:
  name: ova
  namespace: openshift-mtv
spec:
  type: ova
  url: <nfs-sunucu>:/ova
```

✅ **Gerçek çıktı:** OVA provider `Ready`. OVA sunucu pod'u (`ova-<id>`) logunda paylaşımdaki her dosya için `Processing OVA file: /ova/<dosya>.ova`, açılmış dizin için `Processing OVF file: /ova/<dizin>/<ad>.ovf` satırları görüldü.

**Console:** **Migration → Providers for virtualization → Create Provider** → tip seçimi (VMware / Open Virtual Appliance) → bağlantı bilgileri.

---

## 3. Envanter: VM, Ağ ve Disk ID'leri

Plan ve haritalar kaynak nesneleri **ID** ile gösterir. ID'ler MTV envanter API'sinden alınır (Console bunları listeden seçtirir):

```bash
INV=$(oc get route forklift-inventory -n openshift-mtv -o jsonpath='{.spec.host}')
TOKEN=$(oc create token forklift-controller -n openshift-mtv --duration=10m)
PID=$(oc get providers.forklift.konveyor.io ova -n openshift-mtv -o jsonpath='{.metadata.uid}')

curl -sk -H "Authorization: Bearer $TOKEN" "https://$INV/providers/ova/$PID/vms?detail=1"   # VM'ler, diskler, ağlar
curl -sk -H "Authorization: Bearer $TOKEN" "https://$INV/providers/ova/$PID/networks"
curl -sk -H "Authorization: Bearer $TOKEN" "https://$INV/providers/ova/$PID/storages"
# vSphere için yol: /providers/vsphere/<uid>/vms , /networks , /datastores
```

✅ **Gerçek çıktı (VMware'den dışa alınmış bir RHEL CoreOS OVA'sı):** `name: "RHEL CoreOS 9.6"`, `cpuCount: 2`, `memoryMB: 4096`, `firmware: efi`, `osType: rhel8_64Guest`, disk `disk.vmdk` (16 GiB), NIC `Network adapter 1` → ağ `VM Network`. Concern: **`Invalid VM Name`** (adda boşluk ve büyük harf var; Kubernetes adı olamaz).

> `Invalid VM Name` uyarısı engelleyici değildir; Plan'da `targetName` ile geçerli bir ad verilir (bölüm 5).

---

## 4. NetworkMap ve StorageMap

`maps.yaml` (ID'leri envanterden doldurun):

```bash
sed -e 's/REPLACE_ME_SOURCE_PROVIDER/ova/' \
    -e 's/REPLACE_ME_SOURCE_NETWORK_ID/<ağ-id>/' \
    -e 's/REPLACE_ME_SOURCE_STORAGE_ID/<disk-veya-datastore-id>/' maps.yaml | oc apply -f -
```

- Ağ hedefi `type: pod` → VM pod ağına bağlanır. Kurumsal VLAN için [17 — Virtualization](../17-virtualization/README.tr.md) bölüm 9'daki NAD kullanılır: `type: multus, name: <namespace>/<nad-adı>`.
- Disk hedefi `ocs-storagecluster-ceph-rbd-virtualization` → live migration'a uygun RWX Block disk.

✅ **Gerçek çıktı:** `poc-ova-network-map` ve `poc-ova-storage-map` → `Ready=True`.

**Console:** **Migration → NetworkMaps / StorageMaps → Create** — kaynak ve hedefler listeden seçilir. Plan oluşturma sihirbazı haritaları kendisi de üretebilir.

---

## 5. Plan

```bash
oc apply -f namespace.yaml        # hedef namespace: poc-mtv-demo
sed -e 's/REPLACE_ME_SOURCE_PROVIDER/ova/' -e 's/REPLACE_ME_SOURCE_VM_ID/<vm-id>/' \
    -e 's/REPLACE_ME_TARGET_VM_NAME/rhcos-from-ova/' plan.yaml | csplit -s - '/^---$/'   # xx00 = Plan, xx01 = Migration
oc apply -f xx00
oc get plans.forklift.konveyor.io poc-ova-plan -n openshift-mtv \
  -o jsonpath='{range .status.conditions[*]}{.type}={.status} {.message}{"\n"}{end}'
```

- `warm: false` → **cold migration**: kaynak VM kapatılır, disk bir kez kopyalanır. OVA'da sadece cold vardır.
- `targetName`: kaynak ad Kubernetes adına uygun değilse zorunludur.
- Plan oluşturulunca MTV ön kontrol yapar (haritalar eksiksiz mi, hedef namespace var mı, VM'de engelleyici concern var mı).

✅ **Gerçek çıktı:** `Ready=True The migration plan is ready.`

**Console:** **Migration → Plans for virtualization → Create Plan** → kaynak provider → VM seçimi → hedef namespace → haritalar → **Create migration plan**.

---

## 6. Migration ve İzleme

```bash
oc apply -f xx01      # Migration: poc-ova-migration-1
oc get migrations.forklift.konveyor.io poc-ova-migration-1 -n openshift-mtv \
  -o jsonpath='{range .status.vms[0].pipeline[*]}{.name} {.phase} {.error.reasons}{"\n"}{end}'
```

Beklenen pipeline (cold): `Initialize → DiskAllocation → ImageConversion (virt-v2v) → DiskTransferV2v → VirtualMachineCreation`. Başarılı taşımadan sonra hedef namespace'te `VirtualMachine` oluşur (plan `targetPowerState` ayarına göre açık ya da kapalı). Doğrulama:

```bash
oc get vm,pvc -n poc-mtv-demo
virtctl start rhcos-from-ova -n poc-mtv-demo
virtctl console rhcos-from-ova -n poc-mtv-demo      # ya da virtctl vnc / Console sekmesi
```

**Console:** Plan satırında **Start** → ilerleme her VM için adım adım görünür; adıma tıklayınca pod logları açılır.

---

## 7. Canlı Testte Görülenler (OVA + MTV 2.12.9)

### 7.1 OVA taşıması virt-v2v adımında başarısız

✅ **Gerçek çıktı:**

```
Initialize      Completed   (10 sn)
DiskAllocation  Completed   (22 sn)
ImageConversion Running ->  "Guest conversion failed. See pod logs for details."
DiskTransferV2v Pending
VirtualMachineCreation Pending
```

Conversion pod'unun (`<plan>-<vm-id>-xxxxx`, hedef namespace'te) logu:

```
Building command: virt-v2v [-v -x -o kubevirt -os /var/tmp/v2v -on rhcos-from-ova -i ova .]
ova: orig_ova = ., top_dir = /., ova_type = Directory
virt-v2v: error: /./root: Permission denied
```

Pod ortam değişkenlerinde `V2V_diskPath=.` görüldü; OVA paylaşımı pod'a `/ova` olarak bağlı olduğu halde virt-v2v'ye OVA'nın yolu yerine `.` (çalışma dizini `/`) verildi. Envanterde VM kaydının `ovfPath` alanı boştu. Aynı sonuç:

- `.ova` arşiviyle,
- OVA'nın NFS'te kendi dizinine açılmış haliyle (`.ovf` + `.vmdk`; OVA sunucusu `Processing OVF file` diyerek yeni bir VM kaydı üretti),

iki şekilde de alındı. Aynı ortamda, mevcut MTV operatörü kurulmadan (güncellenmeden) önce yapılmış bir OVA taşımasının başarıyla tamamlandığı görüldü. Bu yüzden durum, kullanılan MTV derlemesine özgü bir hata olarak değerlendirildi; Red Hat'in bilinen hatalar listesinde birebir bir kayıt bulunamadı.

### 7.2 `skipGuestConversion: true` diski kopyalamadı

Plan'da `spec.skipGuestConversion: true` ile virt-v2v atlanınca taşıma **32 sn**'de `Succeeded` oldu ve hedefte doğru donanımla bir VM oluştu (2 vCPU, 4Gi, EFI, virtio disk, pod ağı, 16Gi PVC). Ama pipeline'da disk aktarım adımı yoktu ve VM açıldığında:

```
BdsDxe: failed to load Boot0001 "UEFI Misc Device" ...: Not Found
BdsDxe: No bootable option or device was found.
```

yani **disk boş** kaldı. OVA kaynağında bu seçenek kullanılmamalıdır.

### 7.3 POC'de ne yapılmalı

1. Taşımadan önce MTV sürümünü kontrol edin (`oc get csv -n openshift-mtv`) ve sürümün release notes / bilinen hatalar listesine bakın.
2. **Önce tek ve küçük bir test VM'iyle** deneme taşıması yapın; conversion pod'unun ilk satırlarındaki `virt-v2v ... -i <tip> <yol>` komutunda yolun dolu olduğunu doğrulayın.
3. Müşterinin asıl kaynağı genelde vSphere'dir; OVA sadece dışa alınmış imajlar için gereklidir. vSphere provider'ı bu sorundan etkilenmez (disk yolu yerine vCenter/VDDK bağlantısı kullanılır).

---

## 8. vSphere'e Özgü Notlar

- **Warm migration** (`warm: true`): VM çalışırken diskler CBT (Changed Block Tracking) ile ön kopyalanır, kesinti sadece son delta + açılış süresi kadardır. VM'de CBT açık olmalıdır (`ctkEnabled=TRUE`). `cutover` zamanı Migration'da belirtilir.
- **Windows VM'ler:** virt-v2v virtio sürücülerini otomatik kurar; kaynakta Fast Startup / hazırda bekletme kapalı olmalıdır (aksi halde dosya sistemi "dirty" kalır ve dönüşüm başarısız olur).
- **Statik IP:** `preserveStaticIPs: true` ile Windows/Linux misafirlerin statik IP ayarları korunmaya çalışılır; hedef ağın (Multus NAD) aynı VLAN'a çıkması gerekir.
- **Ağ eşleme:** VM'ler kaynakta aynı VLAN'daysa, hedefte 17'deki VLAN NAD'ına (`type: multus`) eşlenmesi uygulamanın IP'sinin değişmemesini sağlar.

---

## 9. Temizlik

```bash
oc delete migrations.forklift.konveyor.io poc-ova-migration-1 -n openshift-mtv
oc delete plans.forklift.konveyor.io poc-ova-plan -n openshift-mtv
oc delete networkmaps.forklift.konveyor.io poc-ova-network-map -n openshift-mtv
oc delete storagemaps.forklift.konveyor.io poc-ova-storage-map -n openshift-mtv
oc delete project poc-mtv-demo
```

> Plan silinince başarısız denemelerden kalan PVC'ler ve conversion pod'ları hedef namespace'te kalabilir; namespace'i silmek bunları da temizler.

---

## Özet Tablo

| Adım | Durum (lab, MTV 2.12.9, OVA) |
|---|---|
| Provider (`ova`, `host`) | ✅ `Ready` |
| Envanter (VM/ağ/disk ID'leri, concern'ler) | ✅ |
| NetworkMap / StorageMap | ✅ `Ready` |
| Plan (`targetName` ile) | ✅ `Ready` |
| Migration — disk ayırma | ✅ |
| Migration — virt-v2v dönüşümü | ❌ `V2V_diskPath=.` (bölüm 7.1) |
| `skipGuestConversion` ile VM oluşturma | ⚠️ VM oluştu, disk boş (bölüm 7.2) |
