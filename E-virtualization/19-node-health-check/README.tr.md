# 19 — Node Kaybında Otomatik Kurtarma (NodeHealthCheck + Self Node Remediation)

> 🇬🇧 [English](README.md) · [← 18 — MTV ile Taşıma](../18-mtv/README.tr.md) · [POC akışı](../../README.tr.md)

Bir worker node aniden çöktüğünde (güç kesintisi, kernel panic, ağ kopması) Kubernetes o node'daki pod'ları ve **VM'leri kendiliğinden başka node'a taşımaz**. Node'un gerçekten kapalı olduğundan emin olamaz; aynı VM iki node'da birden çalışırsa disk bozulabilir. Bu rehber, sağlıksız node'u otomatik olarak **fence eden** (izole edip yeniden başlatan) ve böylece iş yüklerinin güvenle başka node'da açılmasını sağlayan **Node Health Check (NHC)** + **Self Node Remediation (SNR)** kurulumunu anlatır.

Tüm adımlar lab ortamında (OpenShift 4.22, OpenShift Virtualization 4.22.9, NHC 0.12.1, SNR 0.13.1, bare-metal worker'lar) **uçtan uca canlı test edilmiştir**: bir worker node'da kubelet bilerek durduruldu, NHC/SNR node'u yeniden başlattı, üzerindeki VM başka bir node'da açıldı ve node kendiliğinden cluster'a geri döndü.

Senaryo sırası:

1. Kavramlar: neden otomatik fence gerekir
2. Operatörlerin kurulumu
3. NodeHealthCheck kuralı
4. Test VM'i
5. Node'u düşürme ve zaman çizelgesi
6. Ayar önerileri ve dikkat edilecekler
7. Temizlik

---

## 1. Kavramlar

| Bileşen | Rolü |
|---|---|
| **NodeHealthCheck (NHC)** | Seçilen node'ların durumunu izler; bir node belirlenen süre boyunca `Ready=False/Unknown` kalırsa remediation başlatır |
| **Self Node Remediation (SNR)** | Her node'da çalışan bir agent. Kendisi için remediation açılan node, kendini (watchdog / yazılımsal reboot ile) **yeniden başlatır**. Diğer node'lar, güvenli bekleme süresi dolunca node'a `out-of-service` taint'i ekler |
| `out-of-service` taint | Kubernetes'e "bu node kesinlikle kapalı, üzerindeki pod'ları ve volume bağlantılarını temizleyebilirsin" der; StatefulSet pod'ları ve VM'ler ancak bundan sonra başka node'da açılır |
| **Fence Agents Remediation (FAR)** | SNR'nin alternatifi: node'u IPMI/iLO/iDRAC (BMC) üzerinden kapatıp açar. BMC erişimi varsa daha hızlı ve kesindir |

**Neden gerekli?** NHC/SNR olmadan bir node'un kubelet'i ya da ağı koptuğunda VM `Running@<ölü-node>` olarak kalır ve **başka yerde açılmaz**; aynı diski iki kopyanın birden kullanmasını (split-brain) önlemek için KubeVirt node'un fence edildiğinden emin olmak zorundadır.

---

## 2. Operatörlerin Kurulumu

Red Hat kataloğunda (`redhat-operators`) bulunurlar. Aynı adlı paketler community kataloğunda da olabilir; `source: redhat-operators` olduğundan emin olun:

```bash
oc get packagemanifests -n openshift-marketplace -l catalog=redhat-operators | grep -E 'node-healthcheck|self-node-remediation'
oc apply -f operators.yaml          # namespace openshift-workload-availability + 2 Subscription
oc get csv -n openshift-workload-availability
oc get selfnoderemediationtemplate -n openshift-workload-availability
oc get ds self-node-remediation-ds -n openshift-workload-availability
```

✅ **Gerçek çıktı:** `node-healthcheck-operator.v0.12.1 Succeeded`, `self-node-remediation.v0.13.1 Succeeded` (~1 dk). Şablon `self-node-remediation-automatic-strategy-template` ve agent DaemonSet'i (`9/9` node) otomatik oluştu; `SelfNodeRemediationConfig` watchdog olarak `/dev/watchdog` kullanıyor.

**Console:** **Operators → OperatorHub** → "Node Health Check Operator" (SNR bağımlılık olarak otomatik kurulur) → **Install**. Ardından **Compute → NodeHealthChecks → Create**.

---

## 3. NodeHealthCheck Kuralı

`nodehealthcheck.yaml`:

- **Seçici:** sadece worker'lar; `master`/`control-plane` rolleri **açıkça dışlanır**. Control plane node'larının otomatik reboot edilmesi etcd quorum'unu riske atar.
- **`minHealthy: 51%`:** seçilen node'ların yarısından fazlası aynı anda sağlıksızsa (örn. switch arızası) NHC **müdahale etmez**; toplu reboot'un durumu kötüleştirmesini önler.
- **`unhealthyConditions`:** `Ready` 300 sn boyunca `False` ya da `Unknown` kalırsa remediation başlar.

```bash
oc apply -f nodehealthcheck.yaml
oc get nodehealthcheck poc-worker-nhc -o jsonpath='{.status.phase} observed={.status.observedNodes} healthy={.status.healthyNodes}'
```

> **Testte etki alanını küçültmek için** seçiciye tek bir node havuzunun etiketi eklenebilir (dosyadaki yorumlu satır). Lab testinde kural 3 node'luk bir havuzla sınırlandırıldı.

✅ **Gerçek çıktı:** `phase=Enabled observed=3 healthy=3 reason=NHC is enabled, no ongoing remediation`.

---

## 4. Test VM'i

`test-vm.yaml`: `runStrategy: Always` (VM kapanırsa yeniden başlatılır) ve **RWX** disk (`ocs-storagecluster-ceph-rbd-virtualization`). RWX, diskin başka node'a hızlıca bağlanabilmesini sağlar.

```bash
oc create namespace poc-ha-demo
oc apply -f test-vm.yaml
oc get vmi ha-test-vm -n poc-ha-demo -o jsonpath='{.status.nodeName}'
```

Testte VM'in düşürülecek node'a yerleşmesi için `preferredDuringScheduling` node affinity eklendi (`required` kullanılmamalıdır; aksi halde VM başka node'a geçemez).

✅ **Gerçek çıktı:** `VM: Running node=<worker-A>`.

---

## 5. Node'u Düşürme ve Zaman Çizelgesi

Node'un kubelet'i durdurularak "node erişilemez" durumu simüle edildi (makine açık, ama cluster ile iletişimi kopuk — en zor durum):

```bash
oc debug node/<worker-A> --to-namespace=default -- chroot /host \
  systemd-run --on-active=15 --unit=ha-test systemctl stop kubelet.service
```

> Gerçek bir güç kesintisi testi için node BMC/hypervisor üzerinden kapatılabilir; akış aynıdır.

İzleme (her 10 sn): node `Ready` durumu, taint'ler, NHC fazı, SNR nesneleri ve VM'in yeri.

✅ **Gerçek çıktı (kubelet'in durdurulmasından itibaren, 10 sn çözünürlükle):**

| Zaman | Olay |
|---|---|
| 0 sn | kubelet durdu |
| ~65 sn | Node `Ready=Unknown`, `node.kubernetes.io/unreachable` taint'i. VM hâlâ `Running@worker-A` (doğru: node fence edilmeden taşınmaz) |
| ~370 sn | `Unknown` 300 sn'yi geçti → NHC `Remediating`, `SelfNodeRemediation` oluştu, `remediation.medik8s.io/self-node-remediation` taint'i |
| ~375 sn | SNR agent'ı node'u **yeniden başlattı** (node'un önceki açılışının son log satırı remediation başladıktan saniyeler sonra) |
| ~490 sn | SNR güvenli bekleme süresi doldu → `node.kubernetes.io/out-of-service` taint'i |
| ~510 sn | VM yeniden zamanlandı (`Scheduling`) |
| **~525 sn** | **VM `Running@worker-B`**, ardından guest agent bağlandı (`Fedora Linux 44`) |
| ~590 sn | worker-A yeniden açılıp `Ready` oldu; taint'ler kalktı, SNR nesnesi silindi, NHC tekrar `Enabled` |

Node kaybından VM'in başka node'da çalışmasına kadar toplam süre **~8 dk 45 sn**; bunun 5 dakikası NHC'nin `duration: 300s` beklemesidir (bölüm 6).

---

## 6. Ayar Önerileri ve Dikkat Edilecekler

- **Süreyi kısaltmak:** Toplam sürenin en büyük kısmı `unhealthyConditions[].duration` (300 sn) ve SNR'nin güvenli bekleme süresidir. `duration` 60–120 sn'ye indirilebilir, ama kısa ağ dalgalanmalarında gereksiz reboot riski artar. BMC erişimi varsa **Fence Agents Remediation** node'u doğrudan kapattığı için bekleme süresi kısalır.
- **Control plane'i asla dahil etmeyin:** Seçicide `master`/`control-plane` dışlanmalıdır. Bir master'ın kendiliğinden reboot edilmesi etcd'yi azınlıkta bırakabilir.
- **`minHealthy`:** Ağ veya switch arızasında tüm node'lar birden `Unknown` olur; `minHealthy` sayesinde NHC toplu reboot yapmaz.
- **VM'lerde `runStrategy: Always`** (ya da `RerunOnFailure`) olmalı; `Manual` VM'ler başka node'da kendiliğinden açılmaz.
- **RWX disk:** Live migration için de gerekli olan RWX disk, kurtarma sırasında da diskin yeni node'a takılmasını kolaylaştırır.
- **NHC olmadan:** Aynı senaryoda VM, node elle kapatılıp `out-of-service` taint'i elle eklenene kadar `Running@<ölü-node>` olarak kalır.
- **Kalıntılar:** Operatör kaldırıldığında CRD'ler (`*.medik8s.io`), `node-remediation-console-plugin` ConsolePlugin'i ve bazı ClusterRole'ler geride kalır (bölüm 7).

---

## 7. Temizlik

```bash
oc delete project poc-ha-demo
oc delete nodehealthcheck poc-worker-nhc
for s in node-healthcheck-operator self-node-remediation; do
  csv=$(oc get subscriptions.operators.coreos.com $s -n openshift-workload-availability -o jsonpath='{.status.installedCSV}')
  oc delete subscriptions.operators.coreos.com $s -n openshift-workload-availability
  oc delete csv $csv -n openshift-workload-availability
done
oc delete namespace openshift-workload-availability
# OLM'in silmediği kalıntılar:
oc get crd -o name | grep medik8s | xargs -r oc delete
oc delete consoleplugin node-remediation-console-plugin
oc delete clusterrole node-healthcheck-metrics-reader node-healthcheck-operator-aggregation \
  self-node-remediation-ext-remediation self-node-remediation-metrics-reader
oc delete clusterrolebinding node-healthcheck-operator-aggregation
```

✅ **Gerçek çıktı:** Yukarıdaki kalıntıların hepsi operatör kaldırıldıktan sonra cluster'da duruyordu ve elle silindi.

---

## Özet Tablo

| Adım | Sonuç |
|---|---|
| NHC + SNR kurulumu | ✅ ~1 dk, agent tüm node'larda |
| NHC kuralı (worker'lar, master hariç, `minHealthy 51%`) | ✅ `Enabled` |
| Node erişilemez (kubelet durdu) | ✅ ~65 sn'de `Unknown` |
| NHC remediation + SNR reboot | ✅ 300 sn sonra |
| VM başka node'da `Running` | ✅ **~8 dk 45 sn** (toplam) |
| Node'un kendiliğinden geri dönmesi | ✅ ~10 dk (590 sn), taint'ler otomatik kalktı |
