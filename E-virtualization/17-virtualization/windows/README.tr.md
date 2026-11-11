# 17.1 — Windows VM'ler için Boot Source (Golden Image) Hazırlama

> 🇬🇧 [English](README.md) · [← 17 — OpenShift Virtualization](../README.tr.md) · [POC akışı](../../../README.tr.md) · [18 — MTV ile Taşıma →](../../18-mtv/README.tr.md)

OpenShift Virtualization kurulduğunda **Windows template'leri de gelir** (Windows 10/11, Server 2016/2019/2022/2025). Bunlar için ayrıca bir config yapmak gerekmez. RHEL template'lerinden farkı, Windows template'lerinin kullanacağı **işletim sistemi imajının (boot source) gelmemesidir**. Microsoft lisansı nedeniyle Red Hat Windows imajı dağıtamaz; imajı kurum sağlamalıdır.

Bu doküman Windows boot source'u sağlamanın **dört yolunu** anlatır. Dört yol da lab ortamında (OpenShift 4.22.6, OpenShift Virtualization 4.22.9, ODF 4.22.4) **`ocp-poc-win` namespace'inde, Windows Server 2022 Evaluation ISO'su ile uçtan uca canlı test edilmiştir**. Her yolun sonunda, Red Hat'in hazır `windows2k22-server-medium` template'inden VM açılmış ve VM'in Windows Server 2022 olarak açıldığı, ilk açılış ayarlarının uygulandığı ve RDP'nin (3389) erişilebilir olduğu doğrulanmıştır.

> **Test turları:** Dört yöntem ilk turda uçtan uca test edildi. Nihai adlandırmaya geçilen ikinci turda **yöntem 3 (pipeline)** yeniden uçtan uca çalıştırıldı ve oluşan imajdan hazır template ile VM açılıp ilk açılış ayarları ve RDP yeniden doğrulandı (bölüm 4). Yöntem 1, 2 ve 4 aynı manifest'lerle ikinci turda tekrar çalıştırılmadı.

| # | Yöntem | Ne zaman | Canlı test sonucu |
|---|---|---|---|
| 1 | Hazır imajı yükleme (`virtctl image-upload`) | Elde sysprep'li bir Windows diski (qcow2/raw) varsa | ✅ 4,6 GB imaj 5 dk 15 sn'de yüklendi, template'ten VM açıldı |
| 2 | ISO'dan bir kez kurup golden image yapma | Elde sadece ISO varsa, imaj bir kez hazırlanacaksa | ✅ Kurulum + sysprep ~19 dk, template'ten VM açıldı |
| 3 | `windows-efi-installer` Tekton pipeline'ı | ISO'dan tam otomatik, tekrarlanabilir imaj üretimi | ✅ Toplam 31 dk, template'ten VM açıldı |
| 4 | Kurum registry'si + `DataImportCron` | İmajın RHEL'deki gibi otomatik güncel tutulması | ✅ Otomatik içe aktarıldı; yeni sürüm push edilince DataSource kendiliğinden yeni imaja geçti; template'ten VM açıldı |

İçindekiler:

0. Durum tespiti: template var, boot source yok
1. Ortak parçalar: ilk açılış (`unattend.xml`) ve template'ten VM açma
2. Yöntem 1 — Hazır imajı yükleme
3. Yöntem 2 — ISO'dan kurulum + golden image
4. Yöntem 3 — Tekton pipeline (`windows-efi-installer`)
5. Yöntem 4 — Kurum registry'si + `DataImportCron`
6. Boot source'u cluster geneline açma (`openshift-virtualization-os-images`)
7. Canlı testte görülenler / dikkat edilecekler
8. Temizlik

Dosyalar:

| Dosya | İçerik |
|---|---|
| `namespace.yaml` | `ocp-poc-win` namespace'i |
| `common/unattend-first-boot.yaml` | Golden image'dan açılan VM'lerin ilk açılış ayarları (OOBE atlama, şifre, TZ, RDP) |
| `common/add-first-boot-sysprep.json` | Template'ten oluşan VM'e bu ayarları takan JSON patch |
| `1-upload/datasource.yaml` | Yüklenen imajı boot source yapan DataSource |
| `2-iso/autounattend-configmap.yaml` | ISO'dan gözetimsiz kurulum + sysprep dosyası |
| `2-iso/vm-win2k22-installer.yaml` | ISO + boş disk + virtio sürücüleri + sysprep'li kurulum VM'i |
| `2-iso/golden-image.yaml` | Kurulan diskten DataVolume + DataSource |
| `3-pipeline/pipelinerun-win2k22.yaml` | `windows-efi-installer` pipeline çalıştırması (OpenShift'e uyarlanmış) |
| `4-registry-cron/Containerfile` | Windows diskini containerDisk olarak paketleme |
| `4-registry-cron/dataimportcron.yaml` | Registry'den otomatik içe aktarma cron'u |
| `4-registry-cron/test-registry.yaml` | **Sadece test için** ODF PVC'li geçici registry |

---

## 0. Durum Tespiti: Template Var, Boot Source Yok

```bash
# Windows template'leri
oc get template -n openshift -l template.kubevirt.io/type=base | grep -i windows

# Template'lerin beklediği boot source'lar
oc get datasource -n openshift-virtualization-os-images \
  -o custom-columns=NAME:.metadata.name,READY:'.status.conditions[?(@.type=="Ready")].status',REASON:'.status.conditions[?(@.type=="Ready")].reason'

# Hangi boot source'lar otomatik güncelleniyor
oc get dataimportcron -n openshift-virtualization-os-images
```

✅ **Gerçek çıktı:**

- 24 Windows template'i var: `windows2k22-server-medium`, `windows2k22-server-large`, `windows2k22-highperformance-*`, `windows11-desktop-*` vb.
- `win10`, `win11`, `win2k16`, `win2k19`, `win2k22`, `win2k25` DataSource'ları **`Ready=False`, `NotFound`**. Her biri `openshift-virtualization-os-images` altında kendi adıyla bir PVC bekliyor, ama PVC yok.
- `rhel8`, `rhel9`, `fedora`, `centos-stream9` vb. **`Ready=True`**. Bunlar için `rhel9-image-cron`, `fedora-image-cron` gibi `DataImportCron`'lar var; imajlar Red Hat registry'sinden otomatik çekiliyor. Windows için böyle bir cron yok.

Console'da da Windows template kartları "Source not available" olarak görünür ve doğrudan hızlı oluşturma (Quick create) yapılamaz.

```bash
oc process --parameters -n openshift windows2k22-server-medium
```

```
NAME                    VM name                           expression   windows2022-[a-z0-9]{6}
DATA_SOURCE_NAME        Name of the DataSource to clone                win2k22
DATA_SOURCE_NAMESPACE   Namespace of the DataSource                    openshift-virtualization-os-images
```

Template boot source'u parametre olarak aldığı için, **kendi namespace'imizdeki bir golden image'ı gösterebiliriz**. Aşağıdaki testlerin hepsi bu şekilde, paylaşımlı `openshift-virtualization-os-images`'a dokunmadan yapıldı. Cluster geneline açmak için bkz. bölüm 6.

---

## 1. Ortak Parçalar

### 1.1 İlk açılış ayarları (`unattend.xml`)

Golden image'lar `sysprep /generalize /oobe` ile hazırlanır. Bu imajdan açılan her VM ilk açılışta kendi kimliğini (bilgisayar adı, SID) üretir ve OOBE ekranına (dil, yönetici şifresi) düşer. `common/unattend-first-boot.yaml` bu ekranları otomatik geçer:

- Bilgisayar adı: rastgele (`*`)
- Yerel `Administrator` şifresi: `PocDemo2026x` (POC içindir; üretimde her VM'e ayrı ConfigMap/Secret verin)
- Zaman dilimi `Turkey Standard Time`, bölge `tr-TR`, **klavye `en-US`** (bkz. bölüm 7)
- RDP açık + RDP firewall kuralları etkin

VM'e **`sysprep` volume** olarak takılır:

```bash
oc apply -f namespace.yaml
oc apply -f common/unattend-first-boot.yaml
```

### 1.2 Hazır template'ten VM açma (her yöntemin doğrulaması)

```bash
# DATA_SOURCE_NAME: yöntemin ürettiği DataSource (win2k22 / win2k22-uploaded / win2k22-from-iso / win2k22-registry)
oc process -n openshift windows2k22-server-medium \
  -p NAME=win-test -p DATA_SOURCE_NAME=win2k22 -p DATA_SOURCE_NAMESPACE=ocp-poc-win \
  | oc apply -n ocp-poc-win -f -

# Template VM'i "Halted" oluşturur; açmadan önce ilk açılış ayarlarını tak
oc patch vm win-test -n ocp-poc-win --type=json --patch-file common/add-first-boot-sysprep.json
virtctl start win-test -n ocp-poc-win
```

**Doğrulama:**

```bash
# Guest agent üzerinden işletim sistemi, hostname, zaman dilimi
virtctl guestosinfo win-test -n ocp-poc-win

# RDP portu (cluster içinden)
IP=$(oc get vmi win-test -n ocp-poc-win -o jsonpath='{.status.interfaces[0].ipAddress}')
oc run rdp-check -n ocp-poc-win --rm -i --restart=Never --image=registry.access.redhat.com/ubi9/ubi-minimal \
  -- bash -c "timeout 5 bash -c '</dev/tcp/$IP/3389' && echo RDP ACIK"

# Grafik konsol
virtctl vnc win-test -n ocp-poc-win
```

Dışarıdan RDP ile bağlanmak için: `virtctl port-forward vm/win-test 3389:3389 -n ocp-poc-win`, ya da bir `Service` (NodePort/LoadBalancer) veya Multus ile LAN IP'si (bkz. `../README.tr.md` bölüm 9).

**Console:**

1. **Virtualization → Catalog → Template catalog** → "Microsoft Windows Server 2022 VM" (`windows2k22-server-medium`).
2. **Customize VirtualMachine** → **Disk source**: **"PVC (clone PVC)"** ya da **Bootable volume** olarak kendi golden image'ınızı seçin. Boot source cluster geneline açılmışsa (bölüm 6) bu adım gerekmez, kart doğrudan "Source available" olur.
3. **Scripts → Sysprep** → **Edit** → `unattend.xml` alanına `common/unattend-first-boot.yaml` içindeki XML'i yapıştırın.
4. **Create VirtualMachine**.

---

## 2. Yöntem 1 — Hazır İmajı Yükleme

**Ne zaman:** Kurumun elinde zaten hazırlanmış, sysprep'li bir Windows diski varsa. Örnekler: başka bir sanallaştırma platformundan (VMware/Hyper-V) dışa alınmış golden image, ya da yöntem 2/3 ile bir kez üretilip saklanmış imaj.

Testte, yöntem 3'ün ürettiği golden disk önce `virtctl vmexport` ile dışa alındı. Böylece "elimizde hazır bir Windows diski var" durumu gerçekçi şekilde oluşturuldu:

```bash
virtctl vmexport download win2k22-export -n ocp-poc-win --pvc=win2k22 \
  --output=win2k22-golden.img.gz --insecure
```

✅ **Gerçek çıktı:** 20 GB'lık disk **8 dk 45 sn**'de sıkıştırılmış raw (`.img.gz`, 4,6 GB) olarak indi.

**Yükleme:**

```bash
virtctl image-upload dv win2k22-uploaded -n ocp-poc-win \
  --size=25Gi --image-path=./win2k22-golden.img.gz \
  --storage-class=ocs-storagecluster-ceph-rbd-virtualization \
  --access-mode=ReadWriteMany --volume-mode=block --insecure --force-bind

oc apply -f 1-upload/datasource.yaml        # DataSource win2k22-uploaded
```

> CDI `qcow2`, `raw`, `vmdk`, `vhd(x)` ve bunların `.gz`/`.xz` sıkıştırılmış hallerini kabul eder ve raw'a çevirir. VMware'den gelen `.vmdk` doğrudan yüklenebilir. Bir VMware VM'ini bütün olarak taşımak için ise **Migration Toolkit for Virtualization (MTV)** daha uygundur (bkz. [18 — MTV ile Taşıma](../../18-mtv/README.tr.md)).

✅ **Gerçek çıktı:** Yükleme **5 dk 15 sn** sürdü, `DataSource win2k22-uploaded` → `Ready=True`. Template'ten açılan `win-from-upload`:

```
hostname: ADMINAC-BDC0DA2 | timezone: Turkey Standard Time, 10800 | Windows Server 2022 Datacenter Evaluation
RDP 3389 ACIK -> 10.129.2.110 (VM başlatıldıktan 393 sn sonra)
```

**Console:** **Virtualization → Bootable volumes → Add volume** → **Source type: Upload volume** → imaj dosyası → **Volume name** `win2k22-uploaded`, **Preference** `windows.2k22.virtio` → **Save**.

---

## 3. Yöntem 2 — ISO'dan Kurulum + Golden Image

**Ne zaman:** Elde sadece Microsoft ISO'su varsa ve imaj bir kez, kontrollü şekilde hazırlanacaksa. Akış [17 — OpenShift Virtualization](../README.tr.md) bölüm 4–5'teki (ISO → kurulum → golden image) akışın Windows versiyonudur.

### 3.1 ISO'yu yükleme

```bash
# Windows Server 2022 Evaluation (180 gün): https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2022
curl -L -o win2k22-eval.iso 'https://go.microsoft.com/fwlink/p/?LinkID=2195280&clcid=0x409&culture=en-us&country=US'

virtctl image-upload dv win2k22-iso-upload -n ocp-poc-win --size=6Gi \
  --image-path=./win2k22-eval.iso \
  --storage-class=ocs-storagecluster-ceph-rbd-virtualization --insecure --force-bind
```

✅ **Gerçek çıktı:** ISO 5.044.094.976 bayt (`SSS_X64FREE_EN-US_DV9`, bootable). Yükleme **7 dk 25 sn** sürdü.

### 3.2 Kurulum VM'i

`2-iso/vm-win2k22-installer.yaml`:

| Disk | Kaynak | Rol |
|---|---|---|
| `installcdrom` (sata, bootOrder 2) | `win2k22-iso-upload` PVC | Windows kurulum ISO'su |
| `rootdisk` (virtio, bootOrder 1) | 60Gi boş DataVolume (RWX Block) | Windows buraya kurulur |
| `virtiocontainerdisk` (sata cdrom) | `registry.redhat.io/container-native-virtualization/virtio-win-rhel9` | virtio disk/ağ sürücüleri, guest agent (`E:\`) |
| `sysprep` (sata cdrom) | `poc-win2k22-autounattend` ConfigMap | Gözetimsiz kurulum dosyası (`F:\`) |

- Instancetype `u1.large` (2 vCPU / 8Gi), preference `windows.2k22.virtio` (EFI + SecureBoot + TPM, Hyper-V enlightenments).
- virtio-win imajı cluster'ın kendi tanımından alınır: `oc get cm virtio-win -n openshift-cnv -o jsonpath='{.data.virtio-win-image}'`.

`2-iso/autounattend-configmap.yaml`, KubeVirt'in upstream `windows2k22-autounattend` dosyasıdır (`AcceptEula=true` yapıldı). Yaptıkları:

1. Diski EFI/GPT düzeninde bölümler, virtio sürücülerini `E:\amd64\2k22` yolundan yükler, `Windows Server 2022 SERVERDATACENTER` imajını kurar.
2. **Audit** modunda açılır ve `post-install.ps1`'i çalıştırır: virtio guest tools + qemu-guest-agent kurulur, CD çıkarılır.
3. **`sysprep /generalize /oobe`** yapar ve VM'i kapatır. Sonuç golden image olmaya hazır bir disktir.

```bash
oc apply -f 2-iso/autounattend-configmap.yaml
oc apply -f 2-iso/vm-win2k22-installer.yaml
virtctl vnc win2k22-installer -n ocp-poc-win      # kurulum ekranı
```

> ⚠️ **"Press any key to boot from CD or DVD"**: Windows ISO'ları EFI modunda bu istemi yaklaşık 5 sn gösterir. Tuşa basılmazsa ISO'dan açılmaz. VM açılır açılmaz VNC konsolundan bir tuşa basılmalıdır.
>
> ✅ **Gerçek çıktı:** İlk açılışta istem kaçırıldı: `BdsDxe: No bootable option or device was found.` VM yeniden başlatılıp açılışta tuşa basıldı ve **Windows Boot Manager → "Windows Setup [EMS Enabled]"** ekranı geldi. Testte art arda çok sayıda tuş gönderildiği için Boot Manager'ın geri sayımı da durdu ve Enter'a basılması gerekti. Konsolda tek tuşa basmak yeterlidir. Yöntem 3'teki pipeline bu istemi ISO'dan kaldırarak sorunu tamamen çözer.

**Elle (GUI) kurulum yapılacaksa:** `sysprep` volume'u VM'den çıkarın. Kurulumu VNC'den yapın; disk seçim ekranında **"Load driver" → `E:\amd64\2k22`** ile virtio disk sürücüsünü yükleyin. Kurulumdan sonra `E:\virtio-win-guest-tools.exe`'yi çalıştırın ve son olarak şunu çalıştırın:

```
C:\Windows\System32\Sysprep\sysprep.exe /generalize /oobe /shutdown
```

✅ **Gerçek çıktı (gözetimsiz):** Kurulum başladıktan (~11:58) sonra VM sysprep ile kendini kapattı (`12:17:01 Stopped`). Kurulum + güncelleme + sysprep **~19 dk** sürdü.

**Console:** **Catalog → Template catalog → windows2k22-server-medium** → **Customize VirtualMachine**:

- **Boot from CD** işaretle → **CD source**: `win2k22-iso-upload` (ya da URL / Upload).
- **Disk source**: **Blank** (60Gi).
- **"Mount Windows drivers disk"** işaretle (virtio-win CD'si otomatik takılır).
- **Scripts → Sysprep** → `autounattend.xml` alanına ConfigMap'teki XML'i yapıştırın. Elle kurulum yapılacaksa boş bırakın.

### 3.3 Golden image

```bash
oc apply -f 2-iso/golden-image.yaml     # DataVolume + DataSource win2k22-from-iso
```

✅ **Gerçek çıktı:** Template'ten açılan `win-from-iso`:

```
hostname: ADMINAC-CHN6LTE | timezone: Turkey Standard Time, 10800 | Windows Server 2022 Datacenter Evaluation | agent: 110.2.2
RDP 3389 ACIK -> 10.128.2.148 (VM başlatıldıktan 412 sn sonra)
```

(Guest agent 110.2.2 = Red Hat'in virtio-win imajından gelen sürüm.)

**Console:** **Bootable volumes → Add volume → Use existing volume** → PVC `win2k22-installer-rootdisk`.

---

## 4. Yöntem 3 — Tekton Pipeline (`windows-efi-installer`)

**Ne zaman:** Windows imajının **tam otomatik ve tekrarlanabilir** üretilmesi istendiğinde; örneğin her ay güncel ISO'dan yeni imaj üretmek. Pipeline yöntem 2'deki her şeyi kendisi yapar: ISO'yu Microsoft'tan indirir, "press any key" istemini ISO'dan kaldırır, gözetimsiz kurar, sysprep yapar, DataSource'u oluşturur ve geçici kaynakları temizler.

**Ön koşullar:**

- OpenShift Pipelines kurulu (lab testinde: `openshift-pipelines-operator-rh.v1.23.1`).
- Hub resolver açık: `oc get cm resolvers-feature-flags -n openshift-pipelines` → `enable-hub-resolver: "true"`. Pipeline ve task'lar ArtifactHub'daki `kubevirt-tekton-pipelines` kataloğundan çekilir; cluster'ın internete çıkışı olmalıdır. Kapalı ağda pipeline YAML'ı `https://github.com/kubevirt/kubevirt-tekton-tasks/releases` adresinden alınıp cluster'a uygulanır ve `pipelineRef` yerel isimle kullanılır.
- Namespace'te `pipeline` ServiceAccount'u (OpenShift Pipelines otomatik oluşturur; `openshift-*` / `kube-*` namespace'lerinde oluşturmaz).

> 4.22'de SSP operatörü Tekton task/pipeline'larını artık kendisi dağıtmıyor; pipeline bu yüzden ArtifactHub / GitHub release'lerinden alınıyor.

```bash
oc create -f 3-pipeline/pipelinerun-win2k22.yaml
tkn pipelinerun logs -f -n ocp-poc-win        # ya da Console: Pipelines → PipelineRuns
```

> ⚠️ **OpenShift uyarlaması:** Upstream örnek PipelineRun'daki `taskRunSpecs` bloğu (`runAsUser: 107`, `fsGroup: 107`) OpenShift'te SCC'ye takılır.
>
> ✅ **Gerçek çıktı (ilk deneme):** `modify-windows-iso-file` → `PodAdmissionFailed: ... provider pipelines-scc: .spec.securityContext.fsGroup: Invalid value: [107]: 107 is not an allowed group`. Blok kaldırıldı (dosyadaki hali), ikinci denemede sorun çıkmadı.

✅ **Gerçek çıktı (başarılı çalıştırma):**

| Task | Başlangıç → Bitiş (UTC) | Süre |
|---|---|---|
| `import-autounattend-configmaps`, `import-win-iso`, `create-vm-root-disk` | 10:53:30 → 10:53:51 | ~20 sn (ISO CDI ile arka planda indirildi) |
| `modify-windows-iso-file` (ISO indirmeyi bekleme + "press any key" kaldırma) | 10:53:54 → 11:08:24 | 14,5 dk |
| `create-vm` + `wait-for-vmi-status` (gözetimsiz kurulum + sysprep) | 11:08:24 → 11:24:40 | 16 dk |
| `create-datasource-root-disk`, `cleanup-vm`, `delete-imported-iso`, `delete-imported-configmaps` | 11:24:41 → 11:25:01 | 20 sn |
| **Toplam** | | **31 dk**, `Succeeded` |

Sonuç: `DataVolume win2k22` (20Gi, RWX Block) + `DataSource win2k22`. Geçici VM, ISO ve ConfigMap'ler pipeline tarafından silindi.

✅ **İkinci test turu:** Pipeline **22 dk**'da `Succeeded` (ISO indirme + düzenleme ~11 dk, gözetimsiz kurulum + sysprep ~10 dk). Template'ten açılan VM: `Windows Server 2022 Datacenter Evaluation`, `timezone: Turkey Standard Time`, rastgele hostname; **RDP 3389, VM başlatıldıktan 305 sn sonra açık**.

Template'ten açılan `win-from-pipeline`:

```
{"prettyName":"Windows Server 2022 Datacenter Evaluation","versionId":"2022","kernelRelease":"20348"}
hostname: ADMINAC-52MM6AG | timezone: Turkey Standard Time, 10800
RDP 3389 ACIK -> 10.129.2.81 (VM başlatıldıktan 270 sn sonra)
```

VNC'de OOBE ekranı çıkmadan doğrudan kilit ekranı geldi (`30 Eylül Çarşamba`, Türkiye saati). `Administrator` / `PocDemo2026x` ile giriş yapıldı. PowerShell: `fDenyTSConnections=0`, `TermService Running`.

**Parametreler** (`3-pipeline/pipelinerun-win2k22.yaml`):

| Parametre | Değer | Not |
|---|---|---|
| `winImageDownloadURL` | Server 2022 Evaluation linki | Kurumsal lisanslı ISO için iç web sunucusu URL'si verin |
| `acceptEula` | `true` | Microsoft EULA'sının kabulü |
| `preferenceName` | `windows.2k22.virtio` | |
| `autounattendConfigMapName` | `windows2k22-autounattend` | Windows 11: `windows11-autounattend`, 2025: `windows2k25-autounattend` |
| `baseDvName` | `win2k22` | Üretilen golden image'ın adı |

**Console:** **Pipelines → PipelineRuns → Create PipelineRun** (YAML) ya da **Import YAML** ile dosyayı uygulayın. İlerleme görsel olarak (task kutuları) izlenir.

---

## 5. Yöntem 4 — Kurum Registry'si + `DataImportCron`

**Ne zaman:** Windows imajının da RHEL/Fedora gibi **otomatik güncel tutulması** istendiğinde. Kurumun imaj ekibi yeni imajı (örneğin aylık yamalı) registry'ye push eder. Cluster'daki `DataImportCron` belirlenen zamanlarda registry'yi yoklar; yeni bir digest görürse imajı içe aktarır ve DataSource'u yeni imaja yönlendirir. Template'ler her zaman en güncel imajdan VM açar.

### 5.1 containerDisk oluşturma ve registry'ye gönderme

```bash
# Golden disk (yöntem 1/2/3'ten) -> sıkıştırılmış qcow2 -> containerDisk
qemu-img convert -c -O qcow2 win2k22-golden.raw win2k22-golden.qcow2     # .gz KULLANMAYIN (bkz. 5.2)
podman build -t <registry>/windows/win2k22:$(date +%Y%m%d) \
  --build-arg DISK=win2k22-golden.qcow2 -f 4-registry-cron/Containerfile .
podman push <registry>/windows/win2k22:$(date +%Y%m%d)
podman tag  <registry>/windows/win2k22:$(date +%Y%m%d) <registry>/windows/win2k22:latest
podman push <registry>/windows/win2k22:latest
```

`Containerfile`, diski `/disk/` altına qemu kullanıcısına (UID 107) ait olarak koyar. Disk **qcow2 ya da raw** olmalıdır; `.gz`/`.xz` registry kaynağında açılmaz (bkz. 5.2).

> **Test ortamı notu:** Lab ortamının dahili image registry'si `emptyDir` üzerinde çalışıyor. Büyük bir imaj node diskini doldurabilir ve pod yeniden başlarsa imaj kaybolur. Quay kurulu olsa da test hesabı yoktu. Bu yüzden testte kurum registry'sini temsil etmek için `4-registry-cron/test-registry.yaml` ile `ocp-poc-win` içinde ODF PVC'li geçici bir OCI registry kuruldu. Üretimde kurumun Quay/Harbor/Nexus'u kullanılır.

✅ **Gerçek çıktı:** Push (qcow2, 4,6 GiB) **4 dk 35 sn** sürdü. Registry'de `{"name":"windows/win2k22","tags":["20260930-2","20260930","latest"]}`.

### 5.2 `DataImportCron`

```bash
# Registry'nin TLS CA'sı — HEM hedef namespace'te HEM openshift-cnv'de, AYNI adla olmalı (bkz. aşağıdaki uyarı)
for ns in ocp-poc-win openshift-cnv; do
  oc create configmap poc-registry-ca -n $ns --from-file=ca.crt=./registry-ca.crt
done

sed "s|REPLACE_ME_REGISTRY|<registry-host>|" 4-registry-cron/dataimportcron.yaml | oc apply -f -
```

- `schedule: "0 3 * * 1"`: her Pazartesi 03:00'te yeni digest var mı diye bakar. İlk oluşturulduğunda hemen bir kez yoklar.
- `managedDataSource: win2k22-registry`: bu DataSource'u oluşturur ve her zaman en son içe aktarılan imaja yönlendirir.
- `importsToKeep: 2`: eski imajlardan son 2'si tutulur, daha eskileri silinir.
- Kimlik doğrulamalı registry için `source.registry.secretRef` (`accessKeyId` / `secretKey` alanlı Secret).

> ⚠️ **`certConfigMap` iki namespace'te gerekir:** Registry'yi yoklayan job (`initial-job-<cron>-*`) `openshift-cnv` namespace'inde çalışır ve `certConfigMap`'i orada arar. İmajı içe aktaran pod ise ConfigMap'i DataImportCron'un kendi namespace'inde arar.
>
> ✅ **Gerçek çıktı (ilk deneme, ConfigMap sadece `ocp-poc-win`'de):** Cron `UpToDate=False NoDigest`. Yoklama pod'u `ContainerCreating`'de kaldı: `MountVolume.SetUp failed for volume "cdi-cert-vol" : configmap "registry-ca" not found` (`openshift-cnv`). Aynı adlı ConfigMap `openshift-cnv`'de de oluşturulunca yoklama job'ı **5 sn**'de tamamlandı ve digest okundu.

✅ **Gerçek çıktı (ilk sürüm):** Yoklama job'ı digest'i okudu (`sha256:9d11b5c2...`). Cron, adı digest'ten türetilen `win2k22-registry-9d11b5c26657` DataVolume'unu içe aktardı ve `DataSource win2k22-registry` hazır oldu. CDI, ODF üzerinde içe aktarılan diski otomatik olarak **VolumeSnapshot**'a çevirdi (`DataSource src={"snapshot":{"name":"win2k22-registry-9d11b5c26657"}}`); RHEL boot source'ları da aynı şekilde saklanır.

> ⚠️ **containerDisk içinde `.gz` kullanmayın.** İlk push'ta disk `.img.gz` olarak paketlendi. İçe aktarma başarılı göründü, ama bu imajdan açılan VM boot edemedi: `BdsDxe: failed to load Boot0001 "UEFI QEMU HARDDISK" ... No bootable option or device was found`. İçe aktarılan diskin ilk baytları `1f 8b 08 00` (gzip başlığı) çıktı; yani **CDI registry kaynağında arşivi açmıyor, dosyayı olduğu gibi diske yazıyor**. Upload ve HTTP kaynaklarında ise açıyor (yöntem 1). Disk `qemu-img convert -c -O qcow2` ile sıkıştırılmış qcow2'ye çevrildi (20 GiB sanal, 4,6 GiB dosya) ve yeniden push edildi.

### 5.3 Yeni imaj sürümünün otomatik devreye alınması

İmaj ekibinin yeni bir sürüm yayınlaması simüle edildi: qcow2 containerDisk aynı `latest` tag'ine push edildi (yeni digest `sha256:e662cb06...`). Cron Pazartesi 03:00'ü beklemesin diye yoklama elle tetiklendi:

```bash
# DataImportCron spec'i değiştirilemez (webhook: "Cannot update DataImportCron Spec"), bu yüzden
# schedule geçici olarak kısaltılamaz. Yoklamayı hemen yaptırmak için CronJob'dan bir Job oluşturulur:
CJ=$(oc get cronjob -n openshift-cnv -o name | grep win2k22-image-cron)
oc create job --from=$CJ manual-poll-1 -n openshift-cnv
```

✅ **Gerçek çıktı:**

| Adım | Sonuç |
|---|---|
| Yoklama job'ı | **7 sn**'de tamamlandı, yeni digest algılandı |
| Yeni sürümün içe aktarılması | `win2k22-registry-e662cb064934`, **284 sn** |
| `DataSource win2k22-registry` | Otomatik olarak yeni snapshot'a geçti (`src={"snapshot":{"name":"win2k22-registry-e662cb064934"}}`) |
| Eski sürüm | `importsToKeep: 2` gereği tutuldu (`win2k22-registry-9d11b5c26657` snapshot'ı `READY=true`) |
| Cron durumu | `UpToDate=True`, `lastImportedPVC=win2k22-registry-e662cb064934` |

Template'ten açılan `win-from-registry` (yeni sürüm):

```
hostname: ADMINAC-EVE4770 | timezone: Turkey Standard Time, 10800 | Windows Server 2022 Datacenter Evaluation
RDP 3389 ACIK -> 10.128.2.156 (VM başlatıldıktan 319 sn sonra)
```

Template ve VM tanımlarında hiçbir değişiklik yapılmadı; yeni VM'ler otomatik olarak en güncel imajdan açıldı. Mevcut VM'ler etkilenmez, çünkü kendi disklerine sahiptirler.

### 5.4 Cluster genelinde otomatik Windows boot source (HCO)

Yukarıdaki cron namespace'e özeldir. Windows imajının RHEL'deki gibi **cluster genelinde** (`openshift-virtualization-os-images/win2k22`) otomatik güncel tutulması için aynı tanım HyperConverged'a eklenir. Bu işlem hazır Windows template'lerini doğrudan "Source available" yapar:

```yaml
# oc edit hco kubevirt-hyperconverged -n openshift-cnv
spec:
  dataImportCronTemplates:
    - metadata:
        name: win2k22-image-cron
        annotations:
          cdi.kubevirt.io/storage.bind.immediate.requested: "true"
      spec:
        schedule: "0 3 * * 1"
        managedDataSource: win2k22          # template'lerin beklediği DataSource adı
        importsToKeep: 2
        garbageCollect: Outdated
        template:
          spec:
            source:
              registry:
                url: docker://<kurum-registry>/windows/win2k22:latest
                certConfigMap: <registry-ca>   # openshift-cnv ve openshift-virtualization-os-images'da
            storage:
              resources:
                requests:
                  storage: 60Gi
```

> Bu adım paylaşımlı lab cluster'ında **uygulanmadı** (HCO cluster genelidir). Aynı mekanizma yukarıda namespace seviyesinde canlı test edilmiştir.

---

## 6. Boot Source'u Cluster Geneline Açma

Testlerde golden image'lar `ocp-poc-win`'de tutuldu ve template'e parametreyle verildi. Hazır Windows template'lerinin **Console kataloğunda doğrudan "Source available"** görünmesi ve parametre vermeden çalışması için imaj, template'in beklediği yerde olmalıdır: **`openshift-virtualization-os-images/win2k22`** PVC'si. Bunun için üç yol var:

```bash
# a) Yöntem 1'deki gibi doğrudan oraya yükleme
virtctl image-upload dv win2k22 -n openshift-virtualization-os-images --size=60Gi \
  --image-path=./win2k22-golden.img.gz --storage-class=ocs-storagecluster-ceph-rbd-virtualization --insecure

# b) Hazırlanmış bir golden image'ı oraya klonlama
cat <<'EOF' | oc apply -f -
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: win2k22
  namespace: openshift-virtualization-os-images
spec:
  source:
    pvc:
      name: win2k22
      namespace: ocp-poc-win
  storage:
    storageClassName: ocs-storagecluster-ceph-rbd-virtualization
    resources:
      requests:
        storage: 60Gi
EOF

# c) Yöntem 3'te pipeline'ı openshift-virtualization-os-images'da çalıştırma
#    ya da yöntem 4'te HCO dataImportCronTemplates (5.4)
```

Ardından `oc get datasource win2k22 -n openshift-virtualization-os-images` → `Ready=True` olur ve `windows2k22-*` template'leri doğrudan kullanılabilir.

> Bu adım paylaşımlı lab cluster'ında **uygulanmadı**. Bütün cluster kullanıcılarının Windows template'lerini etkileyeceği için platform sahibinin onayıyla yapılmalıdır.

---

## 7. Canlı Testte Görülenler / Dikkat Edilecekler

- **Upstream pipeline örneği OpenShift'e birebir uymuyor:** `taskRunSpecs` (`runAsUser/fsGroup: 107`) → `PodAdmissionFailed`. Kaldırılınca çalıştı (bölüm 4).
- **EFI "Press any key to boot from CD":** Elle ISO kurulumunda VM açılır açılmaz konsoldan tuşa basılmalıdır; yoksa `BdsDxe: No bootable option or device was found`. Pipeline bu istemi ISO'dan kaldırır (bölüm 3.2).
- **Klavye düzeni ve VNC:** İlk denemede `unattend.xml`'de klavye **Türkçe Q** (`InputLocale 041f`) yapıldı. VNC/Console tuşları ABD düzenindeki tuş kodlarıyla gönderildiğinden şifredeki `-` karakteri `*` olarak gitti ve **Console'dan giriş yapılamadı** (`The password is incorrect`). Klavye `en-US` (`0409`) yapılıp bölge/saat `tr-TR` bırakılınca sorun çözüldü. Türkçe klavye isteniyorsa kullanıcı girişten sonra kendi oturumunda ekleyebilir; RDP ile bağlananlar kendi yerel klavyelerini kullanır.
- **RDP firewall kuralı:** `unattend.xml`'de `Networking-MPSSVC-Svc / FirewallGroups / <Group>Remote Desktop</Group>` Server 2022'de **etkisiz kaldı**. RDP servisi açıktı (`fDenyTSConnections=0`) ama "Remote Desktop - User Mode (TCP-In)" kuralları `Enabled=False` idi ve 3389 kapalıydı. `specialize` aşamasında `netsh advfirewall firewall set rule group="remote desktop" new enable=Yes` çalıştıran bir `RunSynchronousCommand` ile çözüldü (`common/unattend-first-boot.yaml`).
- **Windows ağ profili `Public`:** Yeni VM'lerde ağ "Public" kategorisinde gelir ve firewall sadece açıkça izin verilen portları kabul eder. RDP dışındaki servisler (WinRM 5985, SMB 445 vb.) için kural eklenmelidir.
- **Golden image boyutu:** Pipeline kök diski 20Gi oluşturuyor. Template'ler 60Gi disk istediği için klonlanan disk 60Gi olur, ama Windows içindeki `C:` bölümü 20 GB'ta kalır; `Resize-Partition` ile büyütülmesi gerekir. Kalıcı çözüm için pipeline'da kök disk boyutu (`create-vm-root-disk` manifest'i) büyütülebilir ya da unattend'e bölüm genişletme komutu eklenebilir.
- **Windows VM'in `Stopping`'de takılması:** VM UEFI/Boot Manager ekranındayken ACPI kapatma sinyaline cevap vermez ve Windows preference'ındaki uzun bekleme süresi nedeniyle `Stopping`'de kalır. `virtctl stop <vm> --force --grace-period=0` ile durdurulur.
- **`certConfigMap` iki namespace'te:** bkz. bölüm 5.2.
- **containerDisk'te `.gz` yok:** CDI registry kaynağında arşivi açmaz; qcow2/raw kullanın (bölüm 5.2).
- **`DataImportCron` spec'i değiştirilemez:** Schedule/URL değişikliği için cron silinip yeniden oluşturulur. Anlık yoklama için `oc create job --from=cronjob/...` (bölüm 5.3).
- **Lisans:** Testte 180 günlük **Evaluation** sürümü kullanıldı. Üretimde kurumun volume/KMS lisanslı ISO'su ve anahtarı kullanılmalıdır. Aktivasyon unattend'e (`ProductKey`) ya da KMS'e bırakılabilir.

---

## 8. Temizlik

```bash
oc delete namespace ocp-poc-win
oc delete configmap poc-registry-ca -n openshift-cnv
oc delete job manual-poll-1 poc-manual-poll-1 -n openshift-cnv --ignore-not-found
```

(`windows-efi-installer` pipeline'ı kendi geçici VM'ini, ISO'sunu ve ConfigMap'lerini zaten siler. Namespace silindiğinde golden image'lar, test registry'si ve VM'ler de silinir.)
