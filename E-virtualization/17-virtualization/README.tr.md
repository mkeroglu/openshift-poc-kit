# 17 — OpenShift Virtualization POC Rehberi

> 🇬🇧 [English](README.md) · [← 16 — OADP Backup/Restore](../../D-operations/16-oadp/README.tr.md) · [POC akışı](../../README.tr.md) · [17.1 — Windows Boot Source →](windows/README.tr.md)

Bu doküman OpenShift Virtualization (KubeVirt) ile sanal makine yaşam döngüsünü uçtan uca anlatır: template'ten VM, ISO'dan kurulum, golden image, ODF depolama, live migration, snapshot/restore, Multus ile VLAN ağı ve affinity / anti-affinity / nodeSelector ile yerleşim.

Her adım hem **CLI** (`oc` / `virtctl`) hem **Web Console** için verilmiştir. CLI adımlarının tamamı lab ortamında (OpenShift 4.22, OpenShift Virtualization 4.22.9, ODF 4.22, bare-metal worker'lar) **`ocp-poc-virt` namespace'inde uçtan uca canlı test edilmiştir** (iki ayrı test turu); bölümlerdeki "✅ Gerçek çıktı" satırları bu testlerden alınmıştır. Console adımları 4.22 arayüzüne göre yazılmıştır.

Senaryo sırası:

1. Ön koşul kontrolü
2. Operatör kurulumu
3. Template'ler — listeleme, dışa alma (export), template'ten VM, özel template, var olan VM'den template alma
4. ISO ile VM kurulumu (ISO yükleme + kurulum + CD-ROM çıkarma)
5. Golden image (kurulu diskten boot source üretme)
6. ODF depolama — hotplug disk ve online disk büyütme
7. Live migration
8. Snapshot / restore
9. Multus — VM'e VLAN (LAN) arayüzü ekleme
10. Yerleşim — nodeSelector, node affinity, VM affinity / anti-affinity
11. OADP ile VM yedekleme ve geri yükleme
12. Bilinen sınırlamalar / canlı testte görülenler
13. Temizlik
14. Windows VM'ler → ayrı rehber: [17.1 — Windows Boot Source](windows/README.tr.md)

Dosyalar:

| Dosya | İçerik |
|---|---|
| `namespace.yaml` | `ocp-poc-virt` namespace'i |
| `operator-install.yaml`, `hyperconverged.yaml` | Operatör kurulumu (referans) |
| `custom-template.yaml` | `fedora-server-small`'dan türetilmiş POC'a özel template |
| `vm-to-template/vm-to-template.sh` | Var olan VM'den golden image + Template üreten script |
| `vm-to-template/poc-web-golden.yaml` | Script'in canlı testte ürettiği template |
| `vm-from-iso.yaml` | ISO (CD-ROM) + boş diskli VM |
| `golden-image.yaml` | Kurulu diskten DataVolume + DataSource + yeni VM |
| `data-disk-dv.yaml` | Hotplug edilecek veri diski |
| `live-migration.yaml` | `VirtualMachineInstanceMigration` |
| `vm-snapshot.yaml`, `vm-restore.yaml` | Snapshot ve restore |
| `nad-vm-vlan.yaml` | VM'leri kurumsal bir VLAN'a bağlayan `NetworkAttachmentDefinition` (parametrik) |
| `scheduling/*.yaml` | nodeSelector, node affinity, anti-affinity, affinity VM'leri |
| `oadp/namespace.yaml`, `oadp/cross-namespace-clone-rbac.yaml` | Backup testi namespace'i ve namespace'ler arası clone izni |
| `oadp/backup.yaml`, `oadp/restore.yaml` | Velero Backup / Restore |

---

## 1. Ön Koşul Kontrolü

OpenShift Virtualization, VM'leri worker node'lardaki donanım sanallaştırması (KVM) ile çalıştırır.

```bash
# Hangi node'lar VM çalıştırabilir, KVM cihazı var mı?
oc get nodes -l kubevirt.io/schedulable=true \
  -o custom-columns=NODE:.metadata.name,KVM:'.status.allocatable.devices\.kubevirt\.io/kvm'
```

✅ **Gerçek çıktı:** VM çalıştırabilen 6 worker'ın tamamında `KVM: 1k`.

> **Önemli:** Worker'lar bir hypervisor üzerinde VM olarak çalışıyorsa (örn. VMware), ESXi'de ilgili VM'ler için **"Expose hardware assisted virtualization to the guest OS"** açılmalıdır. Açılmazsa `/dev/kvm` olmaz ve VM'ler ya hiç başlamaz ya da sadece çok yavaş yazılımsal emülasyonla (`spec.configuration.developerConfiguration.useEmulation`) çalışır. Bu ayar sadece demo içindir, performans testi yapılmamalıdır.

Depolama tarafında ODF, VM'ler için ayrı bir storage class sunar:

```bash
oc get sc ocs-storagecluster-ceph-rbd-virtualization
oc get storageprofile ocs-storagecluster-ceph-rbd-virtualization -o jsonpath='{.status.claimPropertySets}'
```

✅ **Gerçek çıktı:** İlk tercih `ReadWriteMany + Block`. Live migration için RWX gerektiğinden bu storage class kullanılmalıdır.

---

## 2. Operatör Kurulumu

> Lab ortamında operatör zaten kurulu olduğu için bu adım yeniden uygulanmadı; sadece durum doğrulandı. Aşağıdaki YAML'lar cluster'daki gerçek subscription ile aynıdır (`kubevirt-hyperconverged` / `stable` / `redhat-operators`).

**CLI:**

```bash
oc apply -f operator-install.yaml
oc get csv -n openshift-cnv -w          # kubevirt-hyperconverged-operator ... Succeeded olana kadar bekleyin
oc apply -f hyperconverged.yaml
oc wait hco kubevirt-hyperconverged -n openshift-cnv --for=condition=Available --timeout=15m
```

`virtctl` CLI'ı cluster'ın kendisinden indirilebilir: Console'da **?** → **Command Line Tools** → **virtctl**, ya da `hyperconverged-cluster-cli-download-openshift-cnv.apps.<cluster-domain>` route'u.

**Console:**

1. **Operators → OperatorHub** → "OpenShift Virtualization" → **Install** (varsayılan: `openshift-cnv` namespace, `stable` kanal).
2. Kurulum bitince **Create HyperConverged** → varsayılanlarla **Create**.
3. Sol menüde **Virtualization** bölümü açılır.

**Doğrulama:**

```bash
oc get hco kubevirt-hyperconverged -n openshift-cnv \
  -o jsonpath='{range .status.conditions[*]}{.type}={.status}{"\n"}{end}'
```

✅ **Gerçek çıktı:** `ReconcileComplete=True`, `Available=True`, `Progressing=False`, `Degraded=False`, `Upgradeable=True`.

Namespace'i ve VM'lere enjekte edilecek SSH anahtarını hazırlayın:

```bash
oc apply -f namespace.yaml
ssh-keygen -t ed25519 -N '' -f ./vmkey
oc create secret generic vm-ssh-key -n ocp-poc-virt --from-file=key=./vmkey.pub
```

---

## 3. Template'ler

OpenShift Virtualization, `openshift` namespace'inde hazır VM template'leri ile gelir (RHEL, CentOS Stream, Fedora, Windows ...). Bu template'ler işletim sistemi imajını **boot source** (`DataSource`) üzerinden alır. Boot source'lar `openshift-virtualization-os-images` namespace'inde otomatik olarak güncel tutulur.

### 3.1 Template'leri ve boot source'ları listeleme

```bash
# Hazır (base) template'ler
oc get template -n openshift -l template.kubevirt.io/type=base

# Boot source'lar (template'lerin klonladığı OS imajları)
oc get datasource -n openshift-virtualization-os-images
```

✅ **Gerçek çıktı:** 77 base template; `fedora`, `centos-stream9`, `rhel9`, `rhel10`, `win2k22` vb. 14 DataSource. `fedora`, `rhel9` ve `centos-stream9` `Ready=True`.

### 3.2 Template'i dışa alma (export) ve parametrelerini görme

```bash
# Template'in tamamını YAML olarak alma
oc get template fedora-server-small -n openshift -o yaml > fedora-server-small-template.yaml

# Template'in hangi parametreleri aldığını görme
oc process --parameters -n openshift fedora-server-small
```

✅ **Gerçek çıktı:**

```
NAME                    DESCRIPTION                                          GENERATOR    VALUE
NAME                    VM name                                              expression   fedora-[a-z0-9]{16}
DATA_SOURCE_NAME        Name of the DataSource to clone                                   fedora
DATA_SOURCE_NAMESPACE   Namespace of the DataSource                                       openshift-virtualization-os-images
CLOUD_USER_PASSWORD     Randomized password for the cloud-init user fedora   expression   [a-z0-9]{4}-[a-z0-9]{4}-[a-z0-9]{4}
```

**Console:** **Virtualization → Templates** → proje olarak `openshift` seçin → template'e tıklayın → **YAML** sekmesi. Sağ üstteki **Download** ile dosya olarak indirebilirsiniz.

### 3.3 Template'ten VM oluşturma

**CLI:**

```bash
oc process -n openshift fedora-server-small \
  -p NAME=fedora-from-template -p CLOUD_USER_PASSWORD=Poc-Demo-2026 \
  | oc apply -n ocp-poc-virt -f -

# SSH anahtarını VM'e ekle (cloud-init ile enjekte edilir)
oc patch vm fedora-from-template -n ocp-poc-virt --type=merge -p \
  '{"spec":{"template":{"spec":{"accessCredentials":[{"sshPublicKey":{"source":{"secret":{"secretName":"vm-ssh-key"}},"propagationMethod":{"noCloud":{}}}}]}}}}'

virtctl start fedora-from-template -n ocp-poc-virt     # template'ler VM'i "Halted" oluşturur
oc get dv,vm,vmi -n ocp-poc-virt
```

✅ **Gerçek çıktı:** VM iki test turunda da **19–20 saniyede** `Running` oldu (DataVolume `Succeeded`). Disk sıfırdan indirilmedi, ODF üzerinde boot source snapshot'ından klonlandı. Disk storage profile sayesinde otomatik olarak `ocs-storagecluster-ceph-rbd-virtualization`'da `RWX Block` açıldı.

**Erişim:**

```bash
virtctl ssh fedora@vmi/fedora-from-template -n ocp-poc-virt -i ./vmkey   # SSH (API üzerinden tünel)
virtctl console fedora-from-template -n ocp-poc-virt                      # serial console (çıkış: Ctrl+])
virtctl vnc fedora-from-template -n ocp-poc-virt                          # grafik konsol (yerel VNC istemcisi gerekir)
```

✅ **Gerçek çıktı:** `Fedora release 44`, 1 vCPU, 1.8Gi RAM, 30G `vda`, `qemu-guest-agent: active`.

**Console:**

1. **Virtualization → Catalog** → **Template catalog** sekmesi.
2. Proje: `ocp-poc-virt`; "Fedora VM" (`fedora-server-small`) kartını seçin.
3. Boot source hazır olduğunda kartta "Source available" görünür. **Customize VirtualMachine** ile ad, disk, cloud-init kullanıcı/şifre ve **SSH key** (mevcut secret'ı seçin) ayarlanır.
4. **Create VirtualMachine** ile oluşturun. VM sayfasında **Console** sekmesinden VNC/Serial konsola, **Overview**'dan IP ve node bilgisine ulaşılır.

> **Template yerine InstanceType:** 4.x sürümlerinde önerilen yeni yol **InstanceTypes** sekmesidir. Burada boot volume seçilir, sonra boyut (`u1.small` = 1 vCPU/2Gi, `u1.medium` ...) seçilir. Bölüm 10'daki VM'ler bu yöntemle (`spec.instancetype` + `spec.preference`) oluşturulmuştur.

### 3.4 Özel (kurumsal) template oluşturma

Hazır template dışa alınıp değiştirilerek kuruma özel bir template yapılabilir. Proje namespace'ine konan template'ler Console kataloğunda o projede **"User templates"** olarak görünür.

`custom-template.yaml`, `fedora-server-small`'dan türetilmiştir. Farkları:

- Ad `poc-fedora-small`, namespace `ocp-poc-virt`, görünen ad "POC Fedora Small (SSH key + ODF virt)".
- Label `template.kubevirt.io/type: vm` (kullanıcı template'i). SSP operatörünün yönettiğini belirten `app.kubernetes.io/*` label'ları kaldırıldı.
- Disk her zaman `ocs-storagecluster-ceph-rbd-virtualization` üzerinde açılır.
- Yeni **`SSH_KEY_SECRET`** parametresi eklendi: anahtar VM'e otomatik enjekte edilir.

```bash
oc apply -f custom-template.yaml
oc process --parameters poc-fedora-small -n ocp-poc-virt
oc process poc-fedora-small -n ocp-poc-virt -p NAME=poc-vm-from-custom-template \
  | oc apply -n ocp-poc-virt -f -
virtctl start poc-vm-from-custom-template -n ocp-poc-virt
virtctl ssh fedora@vmi/poc-vm-from-custom-template -n ocp-poc-virt -i ./vmkey
```

✅ **Gerçek çıktı:** Parametre listesinde `SSH_KEY_SECRET ... vm-ssh-key` görüldü. Disk `ocs-storagecluster-ceph-rbd-virtualization`'da açıldı ve ek bir patch gerekmeden `virtctl ssh` ile anahtarla girildi.

**Console:**

- **Virtualization → Templates** → `openshift` projesindeki template'in **⋮** menüsü → **Clone**: yeni ad ve hedef proje (`ocp-poc-virt`) verin. Ardından klonun **YAML** / **Disks** / **Scheduling** sekmelerinden özelleştirin.
- Sıfırdan oluşturmak için **Templates → Create Template** (YAML editörü açılır).

### 3.5 Var olan (kurulmuş) bir VM'den template alma

OpenShift Virtualization'da "VM'den template üret" diye tek adımlık bir API yoktur (cluster'da `VirtualMachineTemplate` benzeri bir CRD bulunmuyor). Bu iş şu akışla yapılır:

1. Kaynak VM'i özelleştir (paket, ayar, hardening).
2. **Genelleştir**: makineye özgü kimlikleri temizle ve VM'i durdur.
3. VM'in root diskini klonlayıp **golden image** (`DataSource`) olarak yayınla.
4. VM tanımından MAC/UUID gibi alanları temizleyip **parametreli bir `Template`** üret. Disk, golden image'dan `sourceRef` ile klonlanır.

Adım 3 ve 4 için `vm-to-template/vm-to-template.sh` script'i hazırlanmıştır.

**1) Özelleştirme** (testte `poc-vm-from-custom-template` VM'i kullanıldı):

```bash
# VM içinde
sudo dnf install -y httpd
echo "<h1>POC golden web sunucusu</h1>" | sudo tee /var/www/html/index.html
sudo systemctl enable --now httpd
echo "POC kurumsal ayar v1" | sudo tee /etc/poc-golden
```

✅ **Gerçek çıktı:** `dnf_rc=0`, `curl localhost` → `<h1>POC golden web sunucusu</h1>`, `httpd enabled`.

**2) Genelleştirme ve durdurma:**

```bash
# VM içinde
sudo cloud-init clean --logs --seed      # yeni VM'lerde cloud-init tekrar çalışsın (hostname, kullanıcı, SSH key)
sudo truncate -s0 /etc/machine-id        # her klon kendi machine-id'sini üretsin
sudo rm -f /etc/ssh/ssh_host_*           # her klon kendi SSH host key'ini üretsin
sudo rm -f /home/fedora/.ssh/authorized_keys
sudo hostnamectl set-hostname localhost
sudo systemctl poweroff
```

```bash
virtctl stop poc-vm-from-custom-template -n ocp-poc-virt
```

> Windows VM'lerde bu adımın karşılığı `sysprep /generalize /oobe /shutdown` komutudur.

**3+4) Golden image + Template üretme:**

```bash
cd vm-to-template
./vm-to-template.sh poc-vm-from-custom-template ocp-poc-virt poc-web-golden "POC Web Sunucusu (httpd, golden)"
```

Script şunları yapar:

- VM durdurulmamışsa hata verip çıkar (tutarsız imaj alınmasın diye).
- Root diski (en küçük `bootOrder`) `poc-web-golden-image` DataVolume'una klonlar ve `DataSource poc-web-golden` olarak yayınlar.
- VM tanımından `firmware.uuid`, `firmware.serial` ve `macAddress` alanlarını, `vm.kubevirt.io/template*` label'larını ve hotplug/veri disklerini çıkarır.
- `NAME`, `CLOUD_USER_PASSWORD` ve `SSH_KEY_SECRET` parametreli `Template` üretir, `poc-web-golden.yaml` olarak kaydeder ve cluster'a uygular.

✅ **Gerçek çıktı:**

```
Root disk: volume=rootdisk pvc=poc-vm-from-custom-template size=30Gi sc=ocs-storagecluster-ceph-rbd-virtualization
datavolume.cdi.kubevirt.io/poc-web-golden-image created
datasource.cdi.kubevirt.io/poc-web-golden created
datavolume.cdi.kubevirt.io/poc-web-golden-image condition met
template.template.openshift.io/poc-web-golden created
```

Üretilen template (`vm-to-template/poc-web-golden.yaml`) incelendi: MAC/UUID yok, disk `sourceRef: DataSource ocp-poc-virt/poc-web-golden` ile klonlanıyor, CPU/bellek/firmware/affinity kaynak VM'den aynen alınmış.

**5) Template'ten yeni VM'ler:**

```bash
oc process poc-web-golden -n ocp-poc-virt -p NAME=poc-web-01 | oc apply -n ocp-poc-virt -f -
oc process poc-web-golden -n ocp-poc-virt -p NAME=poc-web-02 | oc apply -n ocp-poc-virt -f -

# Doğrulama: her VM'de farklı hostname/machine-id olmalı, httpd ve /etc/poc-golden gelmiş olmalı
virtctl ssh fedora@vmi/poc-web-01 -n ocp-poc-virt -i ./vmkey \
  --command 'hostname; cat /etc/machine-id /etc/poc-golden; systemctl is-active httpd; curl -s localhost'
```

✅ **Gerçek çıktı:** `oc process --parameters` → `NAME` (varsayılan `poc-web-golden-[a-z0-9]{6}`), `CLOUD_USER_PASSWORD`, `SSH_KEY_SECRET`. Template'ten açılan iki VM'de:

| | `poc-web-01` | `poc-web-02` |
|---|---|---|
| hostname | `poc-web-01` | `poc-web-02` |
| machine-id | `92a2346b...` | `de0affe7...` |
| machine-id (2. tur) | `298ee517...` | `f29cef4c...` |
| SSH host key | `SHA256:omHiiCKc...` | `SHA256:8bCZoXDD...` |
| MAC | `02:ed:88:63:8b:48` | `02:ed:88:63:8b:49` |
| `/etc/poc-golden` | `POC kurumsal ayar v1` | `POC kurumsal ayar v1` |
| httpd | `active`, `<h1>POC golden web sunucusu</h1>` | `active`, `<h1>POC golden web sunucusu</h1>` |

Özelleştirmeler (paket, servis, ayar dosyası) golden image'dan geldi. Makineye özgü kimlikler (hostname, machine-id, SSH host key, MAC) her VM'de yeniden üretildi; genelleştirme doğru çalıştı. Diskler `ocs-storagecluster-ceph-rbd-virtualization` üzerinde 30Gi olarak açıldı.

> **Template'i başka bir namespace'te kullanmak:** Template ve golden image `ocp-poc-virt`'te dururken başka bir namespace'te VM açılırsa disk klonu `UnauthorizedDataVolumeCreate` hatasıyla bekler (bkz. bölüm 11.2). Kaynak namespace'te clone izni verilmelidir: `oadp/cross-namespace-clone-rbac.yaml`.

**Console karşılığı:**

1. VM'i genelleştirip durdurun (yukarıdaki komutlar, VM **Console** sekmesinden).
2. **Virtualization → Bootable volumes → Add volume** → **Source type: Use existing volume** → PVC `poc-vm-from-custom-template` → ad `poc-web-golden`. Ya da VM → **Snapshots → Take snapshot**, ardından **Add volume → Volume snapshot**.
3. **Virtualization → Templates** → uygun bir base template (örn. `fedora-server-small`) **⋮ → Clone** → proje `ocp-poc-virt`. Klonun **Disks** sekmesinde boot source olarak `poc-web-golden` volume'unu seçin.
4. **Catalog → Template catalog → User templates** üzerinden yeni VM oluşturun.

> **Sadece birebir kopya lazımsa:** `VirtualMachineClone` API'si (`clone.kubevirt.io`) bir VM'i disk ve tanımıyla birlikte doğrudan kopyalar, MAC/SMBIOS değerlerini yeniler. Console: VM → **Actions → Clone**. Bu yöntem template üretmez, tek seferlik kopya içindir.

---

## 4. ISO ile VM Kurulumu

Akış: ISO'yu bir PVC'ye (DataVolume) yükle → VM'i ISO **CD-ROM** + **boş disk** ile oluştur → ISO'dan boot edip diske kur → VM'i durdurup CD-ROM'u çıkar → diskten boot et.

Test için küçük boyutlu ve serial console'dan kurulabildiği için **Alpine Linux 3.24.2 "virt" ISO'su** (66 MB) kullanıldı. RHEL / Windows ISO'larında akış birebir aynıdır, sadece kurulum ekranı farklıdır (Windows için bkz. bölüm 12).

### 4.1 ISO'yu yükleme

**CLI** (`virtctl image-upload`, CDI upload proxy route'u üzerinden):

```bash
curl -LO https://dl-cdn.alpinelinux.org/alpine/latest-stable/releases/x86_64/alpine-virt-3.24.2-x86_64.iso

virtctl image-upload dv alpine-iso -n ocp-poc-virt \
  --size=1Gi \
  --image-path=./alpine-virt-3.24.2-x86_64.iso \
  --storage-class=ocs-storagecluster-ceph-rbd-virtualization \
  --insecure --force-bind
```

✅ **Gerçek çıktı:** `66.00 MiB ... 100.00%` (~35 MiB/s), `Processing completed successfully`. Toplam süre **37 saniye**.

> `--insecure`, CDI upload proxy route'unun sertifikası yerel makinede güvenilir değilse gerekir. `--force-bind`, `WaitForFirstConsumer` storage class'larında PVC'nin hemen bağlanmasını sağlar.

ISO bir web sunucusunda duruyorsa yüklemek yerine doğrudan URL'den de çekilebilir:

```yaml
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: alpine-iso
  namespace: ocp-poc-virt
spec:
  source:
    http:
      url: https://<web-sunucusu>/alpine-virt-3.24.2-x86_64.iso
  storage:
    resources:
      requests:
        storage: 1Gi
```

**Console:**

1. **Virtualization → Bootable volumes** → **Add volume**.
2. **Source type: Upload volume** → ISO dosyasını seçin (ya da **Use URL** ile URL verin).
3. **Volume name** `alpine-iso`, **StorageClass** `ocs-storagecluster-ceph-rbd-virtualization`, **Disk size** `1Gi`.
4. İsteğe bağlı olarak **Preference** (örn. `alpine`) seçin, **Save**. Yükleme ilerlemesi ekranda görünür.

### 4.2 ISO'dan boot eden VM'i oluşturma

`vm-from-iso.yaml` iki disk tanımlar:

- `installation-cdrom`: ISO PVC'si, `cdrom` / `sata`, **`bootOrder: 1`**
- `rootdisk`: 5Gi **boş** (`source.blank`) DataVolume, RWX Block, **`bootOrder: 2`**

```bash
oc apply -f vm-from-iso.yaml
virtctl console alpine-from-iso -n ocp-poc-virt     # kurulum ekranı
```

**Console:**

1. **Virtualization → Catalog** → **Template catalog** → işletim sistemine uygun template'i seçin (örn. "Fedora VM" ya da "RHEL 9 VM").
2. **Customize VirtualMachine** → **Disk source**: **"Boot from CD"** işaretleyin → **CD source**: yüklediğiniz `alpine-iso` PVC'si. **Disk source**: **Blank** (örn. 5Gi).
3. **Create VirtualMachine** → **Console** sekmesinden kurulum ekranına bağlanın.

### 4.3 İşletim sistemini diske kurma

Konsolda `root` ile girilip Alpine kurulum sihirbazı çalıştırılır. Testte aynı adımlar serial console üzerinden cevap dosyasıyla otomatik yapılmıştır:

```sh
# VM konsolunda (ISO'dan açılmış live sistem)
cat > /tmp/answers <<'EOF'
KEYMAPOPTS=none
HOSTNAMEOPTS=alpine-from-iso
INTERFACESOPTS="auto lo
iface lo inet loopback

auto eth0
iface eth0 inet dhcp
"
TIMEZONEOPTS=UTC
PROXYOPTS=none
APKREPOSOPTS="-1"
USEROPTS=none
SSHDOPTS=openssh
ROOTSSHKEY="<vmkey.pub içeriği>"
NTPOPTS=none
DISKOPTS="-m sys /dev/vda"
LBUOPTS=none
APKCACHEOPTS=none
EOF
ERASE_DISKS=/dev/vda setup-alpine -e -f /tmp/answers
poweroff
```

✅ **Gerçek çıktı:** Paketler internetten (pod ağı üzerinden) indirildi, `Installation is complete. Please reboot.`, `SETUP_RC=0`. İkinci turda ISO yükleme 17 sn, kurulum 24 sn sürdü.

### 4.4 CD-ROM'u çıkarma ve diskten boot

> `runStrategy: Always` olan bir VM'de misafir içinden `poweroff` yapılırsa KubeVirt VM'i **yeniden başlatır** (yine ISO'dan açılır). Bu yüzden VM'i `virtctl stop` ile durdurun.

**CLI:**

```bash
virtctl stop alpine-from-iso -n ocp-poc-virt
oc wait vmi alpine-from-iso -n ocp-poc-virt --for=delete --timeout=120s

# CD-ROM diskini ve volume'unu VM tanımından çıkar (index 1 = installation-cdrom)
oc patch vm alpine-from-iso -n ocp-poc-virt --type=json -p \
  '[{"op":"remove","path":"/spec/template/spec/domain/devices/disks/1"},
    {"op":"remove","path":"/spec/template/spec/volumes/1"}]'

virtctl start alpine-from-iso -n ocp-poc-virt
virtctl ssh root@vmi/alpine-from-iso -n ocp-poc-virt -i ./vmkey
```

✅ **Gerçek çıktı:** Diskler: `[{"bootOrder":2,"disk":{"bus":"virtio"},"name":"rootdisk"}]`. SSH ile `alpine-from-iso`, `3.24.2`, `/dev/vda3  3.3G  60.8M  3.1G  2% /`. Sistem artık kurulu diskten açılıyor.

**Console:** VM → **Configuration → Storage** (ya da **Disks**) → `installation-cdrom` satırında **⋮ → Eject CD-ROM** / **Detach** → **Actions → Restart**.

---

## 5. Golden Image (kurulu diskten boot source)

ISO'dan kurulup özelleştirilmiş bir disk "golden image" yapılabilir. Bu imaj kataloğa boot source olarak eklenir ve yeni VM'ler saniyeler içinde bu diskten klonlanır. Kurumların kendi hardening'li RHEL/Windows imajlarını dağıtma yöntemi budur.

`golden-image.yaml` üç kaynak içerir:

1. `DataVolume alpine-golden`: kurulu `alpine-from-iso-rootdisk`'in klonu
2. `DataSource alpine-golden`: bu PVC'yi boot source olarak yayınlar (varsayılan instancetype/preference label'larıyla)
3. `VirtualMachine alpine-from-golden`: `sourceRef` ile bu DataSource'tan açılan yeni VM

```bash
virtctl stop alpine-from-iso -n ocp-poc-virt       # tutarlı kopya için kaynak VM'i durdurun
oc apply -f golden-image.yaml
virtctl start alpine-from-iso -n ocp-poc-virt
virtctl ssh root@vmi/alpine-from-golden -n ocp-poc-virt -i ./vmkey
```

✅ **Gerçek çıktı:** `alpine-golden` ve `alpine-from-golden-rootdisk` **~24 saniyede** `Succeeded`. Yeni VM açıldı: `golden image VM OK: hostname=alpine-from-iso alpine=3.24.2`.

> Hostname'in `alpine-from-iso` olması beklenen bir durumdur: disk birebir kopyalanır. Gerçek golden image'larda imaj öncesinde **genelleştirme** yapılmalıdır: Linux'ta `virt-sysprep` / `cloud-init clean` + machine-id temizliği, Windows'ta `sysprep /generalize`.

**Console:** **Virtualization → Bootable volumes → Add volume → Source type: Use existing volume** (PVC `alpine-from-iso-rootdisk`) ya da **Volume snapshot**. İsteğe bağlı olarak volume **"Set as default boot source"** olarak işaretlenebilir. Oluşan volume **Catalog → InstanceTypes** ekranında seçilebilir hale gelir.

---

## 6. ODF Depolama — Hotplug Disk ve Online Büyütme

### 6.1 Çalışan VM'e disk takma (hotplug)

```bash
oc apply -f data-disk-dv.yaml                  # 10Gi boş, RWX Block DataVolume
oc wait dv fedora-data-disk -n ocp-poc-virt --for=condition=Ready

virtctl addvolume fedora-from-template -n ocp-poc-virt \
  --volume-name=fedora-data-disk --serial=DATADISK01 --persist
```

`--persist` diski VM tanımına da yazar, böylece VM yeniden başladığında disk kalır. Olmazsa disk sadece çalışan instance'a takılır.

```bash
# VM içinde
lsblk -d -o NAME,SIZE,SERIAL
sudo mkfs.xfs /dev/sda && sudo mkdir -p /data && sudo mount /dev/sda /data
```

✅ **Gerçek çıktı:** VM kapatılmadan `sda  10G  datadisk01` göründü. Volume durumu `fedora-data-disk Ready` (`hp-volume-*` attach pod'u ile).

> Hotplug diskler VM içinde **SCSI bus** üzerinden gelir (`/dev/sda`), `virtio` değildir. `--serial` değeri küçük harfe çevrilerek görünür. Disk `/dev/disk/by-id/virtio-*` altında bulunmaz; kalıcı mount için UUID kullanın.

**Console:** VM → **Configuration → Storage** → **Add disk** → **Use existing** / **Blank** → **Save** (VM çalışırken eklenir).

### 6.2 Diski online büyütme

```bash
oc patch pvc fedora-data-disk -n ocp-poc-virt --type=merge \
  -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'

# VM içinde (XFS)
sudo xfs_growfs -d /data
```

✅ **Gerçek çıktı:** PVC `20Gi` oldu. VM içinde, yeniden başlatma olmadan `sda 20G`. `xfs_growfs` sonrası `/dev/sda  20G ... /data`, test dosyası (`veri`) korundu.

**Console:** **Storage → PersistentVolumeClaims** → `fedora-data-disk` → **Actions → Expand PVC**.

---

## 7. Live Migration

Çalışan VM'i kapatmadan başka bir node'a taşır. Node bakımı (`oc adm drain`) sırasında da VM'ler bu şekilde otomatik taşınır. Koşul: VM disklerinin **RWX** olması (burada ODF RBD Block RWX).

```bash
oc get vmi -n ocp-poc-virt \
  -o custom-columns=VM:.metadata.name,NODE:.status.nodeName,MIGRATABLE:'.status.conditions[?(@.type=="LiveMigratable")].status'
```

**Kesintiyi ölçmek için** VM içinde 200 ms aralıkla ping ve zaman damgası sayacı başlatıldı:

```bash
# VM içinde
GW=$(ip route | awk '/default/{print $3}')     # pod ağının gateway'i
nohup ping -i 0.2 -D $GW > /tmp/ping.log 2>&1 &
nohup sh -c 'while true; do date +%T.%N >> /tmp/counter.log; sleep 0.2; done' >/dev/null 2>&1 &
```

**CLI:**

```bash
oc apply -f live-migration.yaml          # ya da: virtctl migrate fedora-from-template -n ocp-poc-virt
oc get vmim -n ocp-poc-virt -w
oc get vmi fedora-from-template -n ocp-poc-virt -o jsonpath='{.status.migrationState}'
```

✅ **Gerçek çıktı:**

| Ölçüm | 1. tur | 2. tur |
|---|---|---|
| Kaynak → hedef | worker-A → worker-B | worker-A → worker-B |
| Mod | `PreCopy` | `PreCopy` |
| Süre | **7 sn** | **9 sn** |
| VM içi ping (0.2 sn aralık) | 209 paket, **0 kayıp** | 246 paket, **0 kayıp** |
| En büyük ping / sayaç boşluğu (switchover) | 0,58 sn | 0,38 / 0,42 sn |
| VM boot zamanı (`uptime -s`) | Değişmedi | Değişmedi |

**Console:** VM → **Actions → Migrate → Compute** (ya da VM listesinde **⋮ → Migrate**). İlerleme **Virtualization → Overview → Migrations** sekmesinde izlenir.

---

## 8. Snapshot / Restore

ODF RBD CSI snapshot'ları üzerinden çalışır. VM **çalışırken** snapshot alınabilir. VM'de `qemu-guest-agent` varsa snapshot öncesi dosya sistemi dondurulur (freeze), böylece uygulama tutarlı bir kopya alınır.

```bash
# VM içinde: snapshot'ta olması gereken veri
echo "snapshot oncesi veri" > ~/onemli-dosya.txt; sync

oc apply -f vm-snapshot.yaml
oc wait vmsnapshot fedora-snap-1 -n ocp-poc-virt --for=condition=Ready
oc get vmsnapshot fedora-snap-1 -n ocp-poc-virt -o jsonpath='{.status.indications}'
```

✅ **Gerçek çıktı:** `phase=Succeeded indications=["GuestAgent","Online"]`. Arkada `ocs-storagecluster-rbdplugin-snapclass` ile 30Gi'lik bir `VolumeSnapshot` oluştu.

**Felaket simülasyonu ve geri dönüş:**

```bash
# VM içinde: veriyi "boz"
rm -f ~/onemli-dosya.txt; echo yanlislik > ~/snapshot-sonrasi.txt

virtctl stop fedora-from-template -n ocp-poc-virt          # restore için VM kapalı olmalı
oc apply -f vm-restore.yaml
oc wait vmrestore fedora-restore-1 -n ocp-poc-virt --for=condition=Ready
virtctl start fedora-from-template -n ocp-poc-virt
```

✅ **Gerçek çıktı:** Restore öncesi home dizininde `snapshot-sonrasi.txt` vardı. Restore sonrası yalnızca `onemli-dosya.txt` kaldı, içeriği `snapshot oncesi veri`. Sonradan oluşturulan dosya kayboldu, silinen dosya geri geldi.

> ⚠️ **Hotplug disk + SELinux: snapshot başarısız olabilir.** İkinci test turunda VM'e 6.1'deki gibi hotplug disk takılıp `/data`'ya bağlanmıştı. Snapshot 5 dakika sonra `Failed` oldu:
>
> ```
> command Freeze failed: "LibvirtError(Code=113, ... guest agent command failed: unable to execute QEMU agent command
> 'guest-fsfreeze-freeze': failed to open /data: Permission denied')"
> ```
>
> **Sebep:** Fedora/RHEL'de `qemu-guest-agent` kısıtlı bir SELinux bağlamında (`virt_qemu_ga_t`) çalışır. Yeni oluşturulan XFS'nin kök dizini etiketsiz (`unlabeled_t`) kaldığı için agent bağlama noktasını açamaz ve dosya sistemini donduramaz.
>
> **Çözüm:** Diski bağladıktan sonra bağlama noktasını etiketleyin (kalıcı bağlama için `/etc/fstab`'a eklemeden önce de geçerlidir):
>
> ```bash
> sudo restorecon -v /data      # Relabeled /data from ...:unlabeled_t:s0 to ...:default_t:s0
> ```
>
> ✅ **Gerçek çıktı:** `restorecon` sonrası snapshot **3 sn**'de `Succeeded` oldu ve bu kez **kök disk + hotplug veri diski** için iki VolumeSnapshot alındı. Restore (**2 sn**) sonrası hem home dizinindeki dosya hem de `/data/test.txt` snapshot anındaki haline döndü.
>
> Snapshot başarısız olursa ona bağlı bir `VirtualMachineRestore` da tamamlanmaz ve VM'i kapalı bırakır; önce `oc get vmsnapshot` ile `Succeeded` olduğu doğrulanmalıdır.

**Console:** VM → **Snapshots** sekmesi → **Take snapshot**. Geri dönmek için VM'i durdurun → snapshot'ın **⋮ → Restore VirtualMachine from snapshot**. Snapshot'tan **ayrı yeni bir VM** de oluşturulabilir: **⋮ → Create VirtualMachine**.

---

## 9. Multus — VM'e VLAN (LAN) Arayüzü

VM'e pod ağına ek olarak kurumsal bir VLAN'dan NIC eklenir. Böylece VM, LAN'daki diğer sunucular gibi doğrudan IP ile erişilebilir olur.

| Parametre | Anlamı | Lab testinde |
|---|---|---|
| `REPLACE_ME_BRIDGE` | Node'larda NNCP ile oluşturulmuş linux-bridge | `br-vm` |
| `REPLACE_ME_VLAN_ID` | VM'lerin bağlanacağı VLAN | kurumsal sunucu VLAN'ı (DHCP var) |
| Kurumsal ağ (route için) | VM'e erişecek istemcilerin bulunduğu ağ(lar) | `<KURUM_AGI>/16` |

### 9.1 Node tarafı (NNCP)

Node'larda, ikinci bir fiziksel NIC'e (switch'te **trunk**) bağlı bir linux-bridge **NMState** (`NodeNetworkConfigurationPolicy`) ile oluşturulur. Lab ortamında bu bridge zaten kurulu olduğu için node ağına dokunulmadı. Örnek NNCP:

```yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: br-vm
spec:
  nodeSelector:
    node-role.kubernetes.io/worker: ""
  desiredState:
    interfaces:
      - name: br-vm
        type: linux-bridge
        state: up
        bridge:
          options:
            stp:
              enabled: false
          port:
            - name: <ikinci-fiziksel-nic>   # switch tarafında VLAN trunk olmalı
              vlan:
                mode: trunk
                trunk-tags:
                  - id-range: {min: 2, max: 4094}
```

```bash
oc get nncp          # <bridge>  Available  SuccessfullyConfigured
```

> NNCP uygulamak node ağ yapılandırmasını değiştirir; yanlış port seçimi node'un ağ bağlantısını kesebilir. Önce tek bir node'da (`nodeSelector` ile) deneyin.

### 9.2 NetworkAttachmentDefinition

```bash
sed -e 's/REPLACE_ME_BRIDGE/br-vm/g' -e 's/REPLACE_ME_VLAN_ID/<vlan-no>/' nad-vm-vlan.yaml | oc apply -f -
```

**Console:** **Networking → NetworkAttachmentDefinitions → Create** → **Network Type: Linux bridge**, **Bridge name** `<bridge>`, **VLAN tag** `<vlan-no>`.

### 9.3 Çalışan VM'e NIC ekleme (hotplug)

> ⚠️ **Önce VM içinde otomatik DHCP'yi kapatın.** Fedora/RHEL'de NetworkManager yeni NIC'e **kendiliğinden** DHCP ile IP alır ("Wired connection 1"). Kurumsal DHCP havuzu statik kullanılan IP'leri de dağıtıyorsa VM, NIC takıldığı anda başka bir sunucunun IP'sini alır. Lab ortamında bu **iki kez** yaşandı: ilkinde başka bir VM'in statik ikinci IP'si (bir VIP), ikincisinde o an kapalı olan başka bir VM'in IP'si dağıtıldı.
>
> ```bash
> # VM içinde, NIC takılmadan ÖNCE
> printf "[main]\nno-auto-default=*\n" | sudo tee /etc/NetworkManager/conf.d/99-no-auto-default.conf
> sudo systemctl reload NetworkManager
> ```

Cluster'ın `vmRolloutStrategy` değeri `LiveUpdate` olduğundan NIC, VM kapatılmadan eklenir; KubeVirt bunu arka planda otomatik bir live migration ile uygular:

```bash
oc get kubevirt -n openshift-cnv -o jsonpath='{.items[0].spec.configuration.vmRolloutStrategy}'   # LiveUpdate
oc patch vm fedora-from-template -n ocp-poc-virt --type=json -p '[
  {"op":"add","path":"/spec/template/spec/domain/devices/interfaces/-","value":{"name":"vlan","bridge":{}}},
  {"op":"add","path":"/spec/template/spec/networks/-","value":{"name":"vlan","multus":{"networkName":"vm-vlan"}}}]'
oc get vmim -n ocp-poc-virt          # kubevirt-workload-update-xxxxx  Succeeded
```

**Console:** VM → **Configuration → Network** → **Add network interface** → **Network**: `vm-vlan`, **Type**: Bridge → **Save**.

✅ **Gerçek çıktı:** Otomatik migration `Succeeded`, VM içinde yeni NIC (`enp2s0`) `UP` ama **IP'siz** (otomatik DHCP kapalı olduğu için).

### 9.4 IP alma (çakışma kontrolüyle) ve routing

VM'de iki arayüz olduğu için VLAN arayüzü de default route alırsa, LAN'dan gelen isteklerin cevabı yanlış arayüzden (NAT'lı pod ağından) çıkar ve bağlantı kurulamaz (asimetrik routing). Bu yüzden VLAN arayüzünden **default route alınmaz**, sadece kurumsal ağlar o arayüzden yönlendirilir. `ipv4.dad-timeout`, IP'yi kullanmadan önce ARP ile çakışma kontrolü yapar:

```bash
# VM içinde
sudo nmcli con add type ethernet ifname enp2s0 con-name vm-vlan \
  ipv4.method auto ipv4.dad-timeout 3000 ipv4.never-default yes \
  ipv4.routes "<KURUM_AGI>/16 <VLAN_GATEWAY>" ipv6.method disabled
sudo nmcli con up vm-vlan
ip -br -4 a show enp2s0
sudo arping -D -c 3 -w 4 -I enp2s0 <alınan-IP>      # 0 = başka cevap veren yok
```

✅ **Gerçek çıktı:**

| Kontrol | Sonuç |
|---|---|
| VLAN arayüzü | DHCP'den `<VLAN>.181/24`; route `<KURUM_AGI>/16 via <VLAN_GATEWAY> dev enp2s0` |
| `arping -D` | `rc=0` (başka cevap veren yok) |
| Kurumsal ağdaki başka bir sunucudan ping | `3 received, 0% packet loss` |
| O sunucudan VM'in LAN IP'sine doğrudan SSH | Başarılı |
| VM'in internet çıkışı | Pod ağından devam etti (`https://quay.io` → `200`) |

İlk test turunda route verilmeden yapılan denemede (VLAN arayüzü default route alınca) dış sunucudan VM'e ping `%100 kayıp` vermişti; route çözümü uygulanınca `0% kayıp` oldu.

> `arping -D` ile çakışma çıkmasa bile, IP'yi statik kullanan cihaz o an **kapalıysa** çakışma daha sonra ortaya çıkar. İkinci turda DHCP'nin verdiği IP'nin, kapalı bir VM'in IP'si olduğu fark edildi ve test biter bitmez bırakıldı (`nmcli con down`). Canlı ortamda VM'lere ya IPAM/DHCP ekibinden **ayrılmış ve rezerve edilmiş** bir blok verilmeli ya da statik IP atanmalıdır.

> Guest agent'ın raporladığı IP (`oc get vmi ... .status.interfaces`) VM içinde yapılan değişikliklerden sonra birkaç saniye eski kalabilir. Doğrulamayı VM içinden (`ip -br a`) yapın.

---

## 10. Yerleşim — nodeSelector, Node Affinity, VM Affinity / Anti-Affinity

VM'ler virt-launcher pod'ları içinde çalıştığı için Kubernetes'in tüm scheduling kuralları `spec.template.spec` altında aynen kullanılır. KubeVirt her virt-launcher pod'una otomatik olarak `vm.kubevirt.io/name=<vm-adı>` label'ını koyar. VM-VM affinity kurallarında bu label kullanılabilir.

Tüm VM'ler instancetype `u1.small` (1 vCPU / 2Gi) + preference `fedora` ile, `fedora` boot source'undan oluşturulur. Dosyalardaki parametreler ortamınıza göre doldurulur:

| Parametre | Anlamı | Lab testinde |
|---|---|---|
| `REPLACE_ME_NODE` | VM'in sabitleneceği node (`oc get nodes`) | 3 node'luk havuzdan biri |
| `REPLACE_ME_NODE_LABEL` | Zorunlu node etiketi (node affinity) | 3 node'luk ikinci bir havuzun rol etiketi |
| `REPLACE_ME_PREFERRED_NODE` | Bu etiketli node'lar içinde tercih edilen node | ikinci havuzdan bir node |
| `REPLACE_ME_POOL_LABEL` | Anti-affinity testinin sınırlandırılacağı **3 node**'luk havuz etiketi | ilk havuzun rol etiketi |

```bash
cd scheduling
FILL='s#REPLACE_ME_NODE_LABEL#<etiket>#; s#REPLACE_ME_PREFERRED_NODE#<node>#; s#REPLACE_ME_POOL_LABEL#<havuz-etiketi>#; s#REPLACE_ME_NODE\b#<node>#'
for f in vm-nodeselector.yaml vm-node-affinity.yaml vm-anti-affinity.yaml; do sed -E "$FILL" $f | oc apply -f -; done
# web-1 Running olduktan sonra:
oc apply -f vm-affinity.yaml

oc get vmi -n ocp-poc-virt -o custom-columns=VM:.metadata.name,PHASE:.status.phase,NODE:.status.nodeName
```

| Dosya | Kural | Beklenen | ✅ Gerçek sonuç |
|---|---|---|---|
| `vm-nodeselector.yaml` | `nodeSelector: kubernetes.io/hostname=<node>` | Sadece o node | `sched-nodeselector` → **verilen node** (iki turda da) |
| `vm-node-affinity.yaml` | **required**: `<etiket>` var; **preferred** (weight 100): `<node>` | Etiketli node'lardan biri, tercihen `<node>` | `sched-node-affinity` → **tercih edilen node** (iki turda da) |
| `vm-anti-affinity.yaml` | 4 VM (`app=poc-web`); required node affinity: 3 node'luk havuz; **required podAntiAffinity** (`topologyKey: kubernetes.io/hostname`) | 3 VM farklı node'larda, 4. VM yerleşemez | 3 VM havuzun üç farklı node'una dağıldı, **biri `ErrorUnschedulable`** (1. turda `web-4`, 2. turda `web-3` — hangisinin açıkta kalacağı rastgele) |
| `vm-affinity.yaml` | **required podAffinity**: `vm.kubevirt.io/name=web-1` ile aynı node | web-1'in node'u | `cache-1` → **web-1 ile aynı node** (iki turda da) |

✅ **Gerçek çıktı (yerleşemeyen VM'in event'i):**

```
0/9 nodes are available: 3 node(s) didn't match Pod's node affinity/selector,
3 node(s) didn't match pod anti-affinity rules, 3 node(s) had untolerated taint(s).
```

(3 master: taint, havuz dışındaki 3 node: node affinity dışı, havuzdaki 3 node: anti-affinity dolu.) Bu davranış **required** kuralın katı olduğunu gösterir. "Mümkünse ayır, değilse yine de çalıştır" isteniyorsa `preferredDuringSchedulingIgnoredDuringExecution` kullanılmalıdır.

**Toleration (referans, uygulanmadı):** Lab ortamındaki worker node'larında taint yok. Paylaşımlı cluster olduğu için test amaçlı taint eklenmedi. VM'leri adanmış (taint'li) node'lara koymak için:

```yaml
spec:
  template:
    spec:
      tolerations:
        - key: dedicated
          operator: Equal
          value: virtualization
          effect: NoSchedule
```

**Console:** VM → **Configuration → Scheduling**:

- **Node selector** → key/value ekleyin.
- **Tolerations** → key/value/effect.
- **Affinity rules** → **Add affinity rule** → **Type**: Node Affinity / Workload (Pod) Affinity / Workload (Pod) Anti-Affinity. **Condition**: Required during scheduling / Preferred (weight). **Topology key**: `kubernetes.io/hostname`. Expression'lar label key/value ile tanımlanır.
- Değişiklikler bir sonraki yerleşimde (restart ya da migration) geçerli olur.

> **Live migration ile ilişkisi:** Migration hedefi de bu kurallara uyar. `nodeSelector` ile tek node'a sabitlenmiş bir VM başka node'a migrate edilemez (`LiveMigratable` olsa bile hedef bulunamaz). Anti-affinity ile tüm node'lar dolmuşsa (`web-1..3` örneği) migration da `Pending` kalır. Node bakımı planlanırken bu dikkate alınmalıdır.

---

## 11. OADP ile VM Yedekleme ve Geri Yükleme

Senaryo: golden template'ten açılmış, içinde kritik veri olan bir VM'in bulunduğu namespace **VM çalışırken** yedeklenir. Ardından namespace tamamen silinir (felaket) ve OADP ile geri yüklenir. Veri bütünlüğü SHA256 ile doğrulanır.

### 11.1 DPA: `kubevirt` ve `csi` plugin'leri

Bu repodaki [16 — OADP](../../D-operations/16-oadp/README.tr.md) rehberiyle kurulan `dpa-odf`'e VM desteği eklenir:

```bash
oc patch dpa dpa-odf -n openshift-adp --type=merge \
  -p '{"spec":{"configuration":{"velero":{"defaultPlugins":["openshift","aws","csi","kubevirt"]}}}}'
oc rollout status deploy/velero -n openshift-adp
oc get dpa dpa-odf -n openshift-adp -o jsonpath='{range .status.conditions[*]}{.type}={.status} {.reason}{"\n"}{end}'
```

✅ **Gerçek çıktı:** Velero init container'ları `openshift-velero-plugin`, `velero-plugin-for-aws`, `kubevirt-velero-plugin`. Velero argümanlarında `--features=EnableCSI`. DPA `Reconciled=True Complete`, `VeleroReady=True`. BSL `dpa-odf-1` `Available`.

- **`kubevirt` plugin'i:** VM, VMI, DataVolume, PVC ve virt-launcher pod'unu birbirine bağlı olarak yedekler. Çalışan VM'lerde snapshot öncesi guest agent ile dosya sistemini dondurur (freeze) ve sonra çözer (unfreeze).
- **`csi` plugin'i:** PVC'leri ODF (Ceph RBD) CSI snapshot'ı ile yedekler.

### 11.2 Test VM'i (ayrı namespace)

```bash
oc apply -f oadp/namespace.yaml                              # ocp-poc-virt-backup
oc create secret generic vm-ssh-key -n ocp-poc-virt-backup --from-file=key=./vmkey.pub
oc apply -f oadp/cross-namespace-clone-rbac.yaml             # golden image başka namespace'te
oc process poc-web-golden -n ocp-poc-virt -p NAME=poc-backup-vm | oc apply -n ocp-poc-virt-backup -f -
```

✅ **Gerçek çıktı (RBAC olmadan):** VM `Stopped` kaldı. Event: `UnauthorizedDataVolumeCreate ... User system:serviceaccount:ocp-poc-virt-backup:default has insufficient permissions in clone source namespace ocp-poc-virt`. `cross-namespace-clone-rbac.yaml` uygulandıktan sonra klon **9 saniyede** `Succeeded` oldu ve VM açıldı.

```bash
# VM içinde kritik veri
echo "backup oncesi kritik veri $(date -u +%FT%TZ)" | sudo tee /var/www/html/kritik.txt; sync
sha256sum /var/www/html/kritik.txt
```

✅ **Gerçek çıktı:** `12c6bd409aca3bb6749e1a15559d82e0059b6ef09e57a40aa3a7fb8e824e95b2`, machine-id `61920a66aee74c70b57134a7fcbb65e5`.

### 11.3 Backup (VM çalışırken)

`oadp/backup.yaml`, CSI snapshot class'ını **Backup üzerindeki annotation ile** belirtir:

```yaml
annotations:
  velero.io/csi-volumesnapshot-class_openshift-storage.rbd.csi.ceph.com: ocs-storagecluster-rbdplugin-snapclass
```

> Velero normalde `velero.io/csi-volumesnapshot-class=true` etiketli VolumeSnapshotClass'ı arar. Lab ortamında bu etiket yoktu. Paylaşımlı VolumeSnapshotClass'ı etiketlemek yerine annotation yöntemi kullanıldı; bu yöntem cluster geneline dokunmaz. Kalıcı kullanım için `oc label volumesnapshotclass ocs-storagecluster-rbdplugin-snapclass velero.io/csi-volumesnapshot-class=true` daha pratiktir.

```bash
oc apply -f oadp/backup.yaml
oc get backups.velero.io poc-vm-backup-1 -n openshift-adp -w
```

> ⚠️ Bu cluster'da CloudNativePG kurulu olduğu için **`oc get backup` CNPG'nin `backups.postgresql.cnpg.io` kaynağını getirir** ve `NotFound` döner. Velero nesneleri için tam adı kullanın: `backups.velero.io`, `restores.velero.io`.

✅ **Gerçek çıktı:**

```
phase=Completed items=91/91 csi=1/1 errors= warnings= start=12:20:48Z end=12:22:21Z
hookStatus: {"hooksAttempted":2}
```

Velero logunda freeze/unfreeze hook'ları: `/usr/bin/virt-freezer --freeze ...` → `Guest agent version is 10.2.2`, `Operation completed successfully`; snapshot sonrası `virt-freezer --unfreeze` → `Operation completed successfully`. Süre **~1,5 dakika**, VM kapatılmadı.

### 11.4 Felaket simülasyonu ve restore

```bash
oc delete namespace ocp-poc-virt-backup          # VM, disk (PVC) ve tüm kaynaklar silinir
oc apply -f oadp/restore.yaml
oc get restores.velero.io poc-vm-restore-1 -n openshift-adp -w
```

✅ **Gerçek çıktı:**

```
phase=Completed items=57/57 errors= warnings=14 start=12:34:39Z end=12:35:22Z
virtualmachine.kubevirt.io/poc-backup-vm   Running
persistentvolumeclaim/poc-backup-vm        Bound   30Gi   RWX   ocs-storagecluster-ceph-rbd-virtualization
```

Restore **43 saniyede** tamamlandı ve VM kendiliğinden açıldı.

✅ **İkinci test turu** (golden template'ten açılan VM ile): backup **27 sn** `Completed 85/85 csi=1/1 hooksAttempted=2`; namespace silindikten sonra restore **27 sn** `Completed 50/50` (13 zararsız uyarı); veri dosyasının SHA256'sı ve machine-id birebir aynı, httpd `active`. Namespace'ler arası klon izni olmadan VM yine `UnauthorizedDataVolumeCreate` ile bekledi; RBAC uygulanınca 42 sn'de açıldı. Disk yeniden klonlanmadı, CSI snapshot'tan geri yüklendi.

**Veri doğrulama (VM içinde):**

```
backup oncesi kritik veri 2026-09-29T12:20:37Z
12c6bd409aca3bb6749e1a15559d82e0059b6ef09e57a40aa3a7fb8e824e95b2  /var/www/html/kritik.txt
machine-id: 61920a66aee74c70b57134a7fcbb65e5
httpd: active
```

SHA256 ve machine-id birebir aynı: aynı VM, verisiyle birlikte geri geldi.

**14 uyarı (zararsız):** Hepsi `could not restore, ... already exists` türündedir. Cluster'da zaten bulunan CRD'ler (`virtualmachines.kubevirt.io`, `datavolumes.cdi.kubevirt.io` ...) ve SCC `kubevirt-controller` ile namespace oluşturulurken OpenShift'in otomatik yarattığı `kube-root-ca.crt`, `openshift-service-ca.crt`, `istio-ca-*` ConfigMap'leri, pipeline RoleBinding'leri ve dockercfg secret'ları için verilir.

**Console:** OADP operatörü Console'a ayrı bir ekran eklemez. **Operators → Installed Operators → OADP → Backup / Restore** sekmelerinden **Create Backup / Create Restore** formları (YAML/form) kullanılır. Durum aynı sekmelerden izlenir.

> **Kapsam notu:** CSI snapshot'lar Ceph içinde tutulur, S3'e sadece Kubernetes nesneleri yazılır. Bu yüzden bu yöntem **aynı cluster'a** geri dönüş için yeterlidir. Ceph'in kendisi kaybedilirse ya da VM başka bir cluster'a taşınacaksa DPA'da `nodeAgent` açılıp Backup'ta `snapshotMoveData: true` (Data Mover) kullanılmalıdır. Bu durumda disk verisi de S3'e (ODF RGW) kopyalanır.

---

## 12. Bilinen Sınırlamalar / Canlı Testte Görülenler

- **DHCP ile IP çakışması (önemli):** Kurumsal VLAN'daki ilk denemede DHCP, test VM'ine başka bir VM'de **statik ikinci IP (VIP)** olarak tanımlı bir adresi verdi (`arping -D` başka bir MAC'ten cevap aldı; dışarıdan SSH başka bir sunucuya düştü). İkinci turda verilen IP de o an kapalı olan başka bir VM'in adresiydi. İki durumda da IP hemen bırakıldı (bkz. 9.3–9.4). **Ders:** Statik IP'ler DHCP sunucusunda rezerve / hariç tutulmalıdır. Canlı ortamda VM'lere LAN IP'si verirken ya IPAM/DHCP ekibinden ayrılmış bir blok alınmalı ya da her IP kullanılmadan önce `arping -D` ile kontrol edilmelidir.
- **Multus NIC otomatik DHCP:** Fedora/RHEL imajlarında NetworkManager yeni eklenen NIC'e kendiliğinden DHCP ile IP alır ("Wired connection 1"). Hotplug öncesinde `no-auto-default=*` ile kapatın (bkz. 9.3).
- **Hotplug disk + SELinux snapshot hatası:** Yeni formatlanan diskin bağlama noktası etiketsiz kalırsa guest agent dosya sistemini donduramaz ve snapshot `Failed` olur; `restorecon -v <bağlama-noktası>` ile çözülür (bkz. 8).
- **`oc get subscription` belirsizliği:** ACM kuruluysa `subscription` kısa adı ACM'in `subscriptions.apps.open-cluster-management.io` kaynağını getirir. OLM için `oc get subscriptions.operators.coreos.com` kullanın.
- **Hotplug diskler SCSI'dır:** `/dev/sdX` olarak görünür, `virtio` değil (bkz. 6.1).
- **`runStrategy: Always` + misafir içi `poweroff`:** VM yeniden başlatılır. Kalıcı kapatma için `virtctl stop` / Console **Stop** kullanın.
- **`virtctl start` sonrası `oc wait vmi`:** VMI nesnesi birkaç saniye sonra oluşur. Hemen `oc wait vmi` çalıştırılırsa `NotFound` döner. `oc wait vm <ad> --for=condition=Ready` kullanın ya da kısa bir bekleme ekleyin.
- **Windows ISO kurulumu:** Windows kurulum ekranı virtio disk/ağ sürücülerini tanımaz. OpenShift Virtualization'ın sağladığı `virtio-win` container disk'i ikinci CD-ROM olarak takılmalıdır (Console'da "Mount Windows drivers disk" kutusu). Canlı test edilmiş ayrıntılar için bkz. [17.1 — Windows Boot Source](windows/README.tr.md).
- **Golden image genelleştirme:** Klonlanan disk hostname, SSH host key, machine-id gibi kimlikleri de taşır (bkz. bölüm 5).
- **Namespace'ler arası disk klonu RBAC ister:** Golden image/template başka namespace'teyse `UnauthorizedDataVolumeCreate` alınır. Kaynak namespace'te hedef namespace'in ServiceAccount'una `datavolumes/source` izni verilmelidir (`oadp/cross-namespace-clone-rbac.yaml`).
- **`oc get backup` belirsizliği:** CloudNativePG gibi `Backup` adlı CRD'si olan operatörler kuruluysa `oc get backup` Velero'yu getirmez. `backups.velero.io` / `restores.velero.io` kullanın.

---

## 13. Temizlik

```bash
oc delete -f scheduling/ --ignore-not-found
oc delete vm alpine-from-golden alpine-from-iso fedora-from-template poc-vm-from-custom-template -n ocp-poc-virt
oc delete vmrestore,vmsnapshot --all -n ocp-poc-virt
oc delete template poc-fedora-small poc-web-golden -n ocp-poc-virt
oc delete namespace ocp-poc-virt ocp-poc-virt-backup
oc delete clusterrole poc-datavolume-cloner
oc delete restores.velero.io poc-vm-restore-1 -n openshift-adp
# Backup'ı S3 verisi ve CSI snapshot'larıyla birlikte silmek için (yoksa TTL ile 72 saat sonra kendiliğinden silinir):
oc create -f - <<'EOF'
apiVersion: velero.io/v1
kind: DeleteBackupRequest
metadata:
  generateName: poc-vm-backup-1-delete-
  namespace: openshift-adp
spec:
  backupName: poc-vm-backup-1
EOF
```

`dpa-odf`'teki plugin eklemesini geri almak için ([16 — OADP](../../D-operations/16-oadp/README.tr.md) rehberindeki orijinal hali):

```bash
oc patch dpa dpa-odf -n openshift-adp --type=merge \
  -p '{"spec":{"configuration":{"velero":{"defaultPlugins":["openshift","aws"]}}}}'
```

---

## 14. Windows VM'ler

Windows template'leri kurulumla birlikte gelir, ancak Microsoft lisansı nedeniyle **Windows boot source'u (imaj) gelmez**; kurumun sağlaması gerekir. Dört yöntem (hazır imajı yükleme, ISO'dan kurulum, Tekton pipeline, registry + `DataImportCron`) Windows Server 2022 ile canlı test edilmiş ve ayrı bir rehberde toplanmıştır: **[17.1 — Windows Boot Source](windows/README.tr.md)**.

---

## Özet Tablo

| Senaryo | CLI | Console | Canlı test |
|---|---|---|---|
| Template listeleme / export / parametreler | `oc get template`, `oc get -o yaml`, `oc process --parameters` | Virtualization → Templates | ✅ |
| Template'ten VM | `oc process ... \| oc apply` | Catalog → Template catalog | ✅ 20 sn'de Running |
| Özel template | `custom-template.yaml` | Templates → Clone | ✅ SSH key parametresiyle |
| Var olan VM'den template | `vm-to-template.sh` | Bootable volumes + Templates → Clone | ✅ 2 VM açıldı; özelleştirmeler geldi, kimlikler yenilendi |
| ISO yükleme | `virtctl image-upload` | Bootable volumes → Upload | ✅ 66 MB / 37 sn |
| ISO'dan kurulum | `vm-from-iso.yaml` + console | Boot from CD | ✅ Kurulum + CD-ROM çıkarma + diskten boot |
| Golden image | `golden-image.yaml` | Bootable volumes | ✅ |
| Hotplug disk / online büyütme | `virtctl addvolume`, `oc patch pvc` | Storage → Add disk / Expand PVC | ✅ 10→20Gi, veri korundu |
| Live migration | `virtctl migrate` / `VirtualMachineInstanceMigration` | Actions → Migrate | ✅ 7 sn, 0 paket kaybı |
| Snapshot / restore | `VirtualMachineSnapshot` / `VirtualMachineRestore` | Snapshots sekmesi | ✅ Online + guest agent freeze |
| Multus VLAN NIC | `nad-vm-vlan.yaml` + patch (hotplug) | Network → Add interface | ✅ LAN IP'sine doğrudan SSH |
| nodeSelector / node affinity | `scheduling/*.yaml` | Scheduling sekmesi | ✅ |
| VM affinity / anti-affinity | `scheduling/*.yaml` | Scheduling → Affinity rules | ✅ 4. VM Unschedulable |
| OADP ile VM backup / restore | `oadp/backup.yaml`, `oadp/restore.yaml` | Installed Operators → OADP | ✅ Çalışırken backup (freeze), namespace silindi, restore 43 sn, SHA256 aynı |
| Windows boot source (4 yöntem) | `windows/` | Bootable volumes, Template catalog, Pipelines | ✅ Dört yöntemde de hazır `windows2k22-server-medium` template'inden VM açıldı, RDP erişilebilir |
