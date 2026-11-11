# 10 — Multus CNI — Macvlan Demo Rehberi

> 🇬🇧 [English](README.md) · [← 09 — Mikro-Segmentasyon](../09-micro-segmentation/README.tr.md) · [POC akışı](../../README.tr.md) · [11 — Güvenlik Testleri →](../11-security-tests/README.tr.md)

Bu doküman, bir pod'a **ikinci bir ağ arayüzü** (gerçek LAN'dan, node'un fiziksel ağıyla aynı segmentten bir IP alan) eklemek için Multus CNI + macvlan kullanımını anlatır. Tüm adımlar lab ortamında (OpenShift 4.22, **OVNKubernetes**, bare-metal worker'lar) **`ocp-poc` namespace'inde, node'ların bulunduğu gerçek LAN'da uçtan uca canlı test edilmiştir**. Kullanılan IP bloğu canlıya geçmeden önce `arping` ile taranıp boş olduğu doğrulanmıştır. Çıktılarda lab ağının adresleri `<LAN>.241` gibi kısaltılmıştır (`<LAN>` = node'ların /24 subnet'i, `<UZAK>` = router arkasındaki başka bir subnet).

Senaryo sırası:

1. Multus/macvlan nedir, ne işe yarar
2. Ön koşul kontrolü (Multus zaten aktif mi, hangi node interface'i kullanılacak)
3. `NetworkAttachmentDefinition` oluşturma (macvlan + whereabouts IPAM)
4. Pod'u ikinci arayüzle çalıştırma
5. Doğrulama (aynı node, farklı node)
6. Farklı subnet'ten (router arkasından) erişim — `rp_filter` ve çözümü
7. Bilinen sınırlamalar / sık yapılan hatalar
8. Temizlik

---

## 1. Multus / Macvlan Nedir?

- **Multus**, bir pod'a birden fazla ağ arayüzü (NIC) bağlamayı sağlayan bir "meta-CNI" eklentisidir. OpenShift'te varsayılan olarak zaten kurulu ve aktiftir — Kubernetes'in standart tek-arayüz (`eth0`, cluster ağı) modelini kırmadan, isteyen pod'lara **ek** arayüz(ler) tanımlamanıza izin verir.
- **Macvlan**, bu ek arayüzlerden biri için kullanılan bir CNI plugin'idir: pod'un ek arayüzüne, node'un **fiziksel** NIC'iyle aynı L2 segmentten (gerçek LAN) kendi MAC + IP adresini verir. Sonuç: pod, cluster'ın SDN'inden (OVN) bağımsız olarak, sanki o fiziksel switch'e doğrudan takılmış bir cihazmış gibi LAN'daki diğer makinelerle doğrudan konuşabilir.
- Tipik kullanım alanları: legacy/on-prem uygulamaların gerçek LAN IP'sine ihtiyaç duyması, VM/NFV benzeri workload'lar, multicast/broadcast gerektiren protokoller, SDN'in NAT'ladığı senaryolarda gerçek kaynak IP'nin görünmesi gereken durumlar.

> **Önemli:** Macvlan, cluster'ın normal `eth0` (OVN) arayüzünün **yerine değil, yanına** eklenir. Pod yine normal Service/Route/NetworkPolicy ile cluster içi trafiğini `eth0` üzerinden yürütmeye devam eder; macvlan sadece ek bir yol açar.

---

## 2. Ön Koşul Kontrolü

**a) Multus / multi-network zaten aktif mi?**

```bash
oc get network.operator cluster -o jsonpath='{.spec.disableMultiNetwork}{"\n"}'
# Beklenen: false (ya da alan hiç yoksa varsayılan zaten false = aktif)
```

OpenShift'te bu **varsayılan olarak açık** gelir — ayrı bir operatör kurulumu gerekmez.

**b) Hangi node interface'i "master" olarak kullanılacak?**

Bu adım kritiktir ve **OVNKubernetes'e özgü bir tuzak** içerir (aşağıda 6. bölümde detaylı). Node'daki fiziksel NIC'i bulun:

```bash
oc debug node/<node-adi> -- chroot /host ip -br link show
```

OVNKubernetes'in yönettiği node'larda birincil fiziksel NIC (örn. `eno1`), **OVS `br-ex` bridge'ine enslave edilmiş** durumdadır — yani doğrudan `eno1`'i macvlan master olarak vermek çoğu zaman beklendiği gibi çalışmaz. Bunun yerine master olarak **`br-ex`** kullanılmalıdır:

```bash
oc debug node/<node-adi> -- chroot /host ip -br addr show br-ex
# br-ex arayüzünün node'un gerçek LAN IP'sini taşıdığını doğrulayın
```

**c) IPAM için whereabouts kurulu mu?**

```bash
oc get crd ippools.whereabouts.cni.cncf.io
```

Bu CRD OpenShift'in Multus paketiyle birlikte gelir; ayrı kurulum gerekmez. `whereabouts`, statik IP çakışmalarını cluster genelinde (birden fazla node'da aynı anda pod başlatılsa bile) önleyen bir IPAM plugin'idir — macvlan gibi "gerçek LAN'dan IP alan" senaryolarda **şiddetle önerilir** (aksi halde iki pod aynı IP'yi alıp LAN'da IP çakışmasına yol açabilir).

---

## 3. NetworkAttachmentDefinition Oluşturma

`NetworkAttachmentDefinition` (NAD), pod'lara referans vereceğiniz ek ağı tanımlayan namespaced bir CRD'dir.

```yaml
# macvlan-nad.yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: macvlan-lan
  namespace: REPLACE_ME_NAMESPACE
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "type": "macvlan",
      "master": "br-ex",
      "mode": "bridge",
      "ipam": {
        "type": "whereabouts",
        "range": "REPLACE_ME_CIDR",
        "range_start": "REPLACE_ME_IP_START",
        "range_end": "REPLACE_ME_IP_END",
        "gateway": "REPLACE_ME_GATEWAY"
      }
    }
```

Doldurulması gerekenler (ağ ekibinden teyit alın — **gerçek LAN'da kullanılmayan ve DHCP havuzunda olmayan** bir bloktur, aksi halde IP çakışması olur):

| Placeholder | Anlamı | Lab testinde |
|---|---|---|
| `REPLACE_ME_NAMESPACE` | Pod'ların namespace'i | `ocp-poc` |
| `REPLACE_ME_CIDR` | Node'ların (br-ex) bulunduğu LAN subnet'i | `<LAN>.0/24` |
| `REPLACE_ME_IP_START` / `REPLACE_ME_IP_END` | Pod'lara verilecek, LAN'da boş blok | `<LAN>.241` – `<LAN>.245` |
| `REPLACE_ME_GATEWAY` | LAN'ın default gateway'i | `<LAN>.1` |

```bash
# Kolay doldurma örneği:
sed -e 's/REPLACE_ME_NAMESPACE/ocp-poc/' -e 's#REPLACE_ME_CIDR#192.168.10.0/24#' \
    -e 's/REPLACE_ME_IP_START/192.168.10.241/' -e 's/REPLACE_ME_IP_END/192.168.10.245/' \
    -e 's/REPLACE_ME_GATEWAY/192.168.10.1/' macvlan-nad.yaml | oc apply -f -
```

**IP bloğunu canlıya geçmeden önce boş olduğunu doğrulama** (ICMP tek başına yetmez, cihaz kapalı olabilir — L2 seviyesinde ARP tablosu + `arping` daha güvenilir):

```bash
# Node üzerinden mevcut ARP tablosunu kontrol et
oc debug node/<node-adi> --to-namespace=default -- chroot /host ip neigh show dev br-ex

# Aday aralığı arping ile L2 seviyesinde sorgula (cevap yoksa boş kabul edilir)
oc debug node/<node-adi> --to-namespace=default -- chroot /host bash -c '
for i in $(seq 241 245); do
  echo "<LAN>.$i -> $(arping -c2 -w2 -I br-ex <LAN>.$i | grep -ci "reply from") cevap"
done'
```

✅ **Gerçek çıktı:** Beş adayın beşi için de `0 cevap` → blok boş.

> `--to-namespace=default`: `oc debug node` varsayılanda mevcut proje namespace'inde geçici bir pod açar. O namespace silinmişse `unable to get namespace` hatası verir; namespace'i açıkça vermek bunu önler.

```bash
oc apply -f macvlan-nad.yaml
```

> **`mode: bridge`** seçildi çünkü aynı fiziksel arayüze bağlı birden fazla macvlan pod'unun **birbiriyle de** konuşabilmesini sağlar. `mode: private`/`vepa` gibi diğer modlar pod-pod iletişimini kısıtlar/switch'e bağımlı kılar — LAN demosu için `bridge` en sorunsuz seçenektir.

---

## 4. Pod'u İkinci Arayüzle Çalıştırma

Pod'un `metadata.annotations` alanında NAD'a referans verilir:

```yaml
# macvlan-test-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: macvlan-test
  namespace: REPLACE_ME_NAMESPACE
  annotations:
    k8s.v1.cni.cncf.io/networks: macvlan-lan
spec:
  containers:
    - name: test
      image: busybox:1.36
      command: ["sleep", "3600"]
```

```bash
oc apply -f macvlan-test-pod.yaml
oc wait --for=condition=Ready pod/macvlan-test -n REPLACE_ME_NAMESPACE --timeout=60s
```

---

## 5. Doğrulama (gerçek test çıktıları)

Testte iki pod **farklı node'lara** (`spec.nodeName`) yerleştirildi; böylece pod'lar arası trafik fiziksel LAN/switch üzerinden geçti.

**a) Pod'a atanan ek arayüz ve IP:**

```bash
oc exec -n ocp-poc macvlan-test -- ip addr show net1
```

✅ **Gerçek çıktı:**

```
3: net1@if14: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue qlen 1000
    link/ether ce:0a:ab:f9:29:0f brd ff:ff:ff:ff:ff:ff
    inet <LAN>.241/24 brd <LAN>.255 scope global net1
```

| Pod | Node | `eth0` (OVN) | `net1` (macvlan/LAN) |
|---|---|---|---|
| `macvlan-test` | worker-A | `10.129.x.x` | `<LAN>.241/24` |
| `macvlan-test-2` | worker-B | `10.131.x.x` | `<LAN>.242/24` |

`eth0` cluster (OVN) IP'sini taşımaya devam ediyor; `net1` whereabouts'tan gelen gerçek LAN IP'sini taşıyor — iki arayüz bir arada, biri diğerinin yerini almıyor. Pod'un **default route'u `eth0`'da** kalır; `net1` sadece kendi subnet'i için bir route ekler.

**b) Aynı NAD'a bağlı iki pod'un (farklı node'larda) macvlan üzerinden birbirine ulaşması:**

```bash
# Dinleyiciyi arka planda başlat (çıktılar yönlendirilmezse oc exec oturumu kapanmaz ve komut takılır)
oc exec -n ocp-poc macvlan-test-2 -- sh -c "nc -l -p 5000 > /tmp/out.txt 2>/dev/null </dev/null &"
oc exec -n ocp-poc macvlan-test   -- sh -c "echo hello | nc -w3 <macvlan-test-2 net1 IP> 5000"
oc exec -n ocp-poc macvlan-test-2 -- cat /tmp/out.txt
```

✅ **Gerçek çıktı:** `hello-capraz-node` — farklı node'lardaki iki pod (`<LAN>.241` → `<LAN>.242`) fiziksel LAN üzerinden TCP ile iletişim kurdu. (Önceki bir test turunda aynı node üzerindeki iki pod da sorunsuz konuştu.)

> **Not:** busybox'ın `ping`'i bu SCC altında `permission denied (are you root?)` veriyor (ICMP raw socket, restricted SCC'de kapalı) — bağlantı testini `nc` (TCP) ile yapın.

---

## 6. Farklı Subnet'ten (Router Arkasından) Erişim — `rp_filter` ve Çözümü

Aynı LAN'daki cihazlar pod'un `net1` IP'sine sorunsuz erişir. Ama **router arkasındaki başka bir subnet'ten** gelen istekler varsayılanda **cevapsız kalır**.

✅ **Gerçek çıktı (sorun):** `<UZAK>` subnet'indeki bir sunucudan `nc <LAN>.241 6000` → `Ncat: TIMEOUT`. Pod'da `cat /proc/sys/net/ipv4/conf/net1/rp_filter` → `1`.

**Kök neden:** Pod'un default route'u `eth0` (OVN) üzerindedir. `<UZAK>`'tan `net1`'e gelen paketin cevabı `eth0`'dan gitmek zorunda kalır; `rp_filter=1` (strict reverse-path filtering) bu asimetriyi görüp paketi **sessizce düşürür**.

### Çözüm (önerilen): IPAM'e route eklemek — node değişikliği gerektirmez

Pod'a "`<UZAK>` subnet'ine `net1`'in gateway'i üzerinden git" route'u verilir. Böylece gelen paket ile cevap aynı arayüzü kullanır ve `rp_filter` paketi kabul eder. `macvlan-nad-routed.yaml`:

```json
"ipam": {
  "type": "whereabouts",
  "range": "REPLACE_ME_CIDR",
  "range_start": "REPLACE_ME_IP_START",
  "range_end": "REPLACE_ME_IP_END",
  "gateway": "REPLACE_ME_GATEWAY",
  "routes": [
    { "dst": "REPLACE_ME_REMOTE_CIDR", "gw": "REPLACE_ME_GATEWAY" }
  ]
}
```

`REPLACE_ME_REMOTE_CIDR`: pod'a erişecek istemcilerin bulunduğu subnet(ler). Birden fazla subnet için listeye yeni satırlar eklenir; kurumun tüm iç ağı için örn. `10.0.0.0/8` verilebilir.

```bash
oc apply -f <doldurulmuş macvlan-nad-routed.yaml>
# Pod annotation'ı: k8s.v1.cni.cncf.io/networks: macvlan-lan-routed  (pod yeniden oluşturulmalı)
oc exec -n ocp-poc macvlan-test -- ip route
```

✅ **Gerçek çıktı (çözüm):**

```
default via 10.129.2.1 dev eth0
<UZAK>.0/24 via <LAN>.1 dev net1            <-- IPAM'den gelen route
<LAN>.0/24 dev net1 scope link src <LAN>.241
```

`<UZAK>` subnet'indeki sunucudan: `nc <LAN>.241 6000` → **`pod1-cevap`**, `ping <LAN>.241` → **`3 packets transmitted, 3 received, 0% packet loss`**. Node'lara dokunulmadı, reboot gerekmedi.

### Alternatifler (önerilmez)

- **`tuning` CNI plugin ile `rp_filter`'ı gevşetmek:** OpenShift bunu admission seviyesinde engeller; pod `ContainerCreating`'de kalır:
  ```
  plugin type="tuning" failed (add): Sysctl net.ipv4.conf.IFNAME.rp_filter is not allowed.
  Only the following sysctls are allowed: [^net.ipv4.conf.IFNAME.accept_redirects$ ...]
  ```
- **MachineConfig ile node genelinde `net.ipv4.conf.default.rp_filter=2`:** Çalışır, ama MCO ilgili node'ları sırayla **reboot** eder ve o node'daki bütün pod'ları etkiler. Platform ekibinin onayı gerekir; yukarıdaki route çözümü varken gerek yoktur.

---

## 7. Bilinen Sınırlamalar / Sık Yapılan Hatalar (canlı testte doğrulanmış)

- **`eno1` değil `br-ex` — OVNKubernetes'e özgü tuzak:** Bu cluster'da birincil fiziksel NIC OVS `br-ex` bridge'ine enslave edilmiş durumda. Master olarak doğrudan fiziksel NIC adını (`eno1`) verirseniz NAD **oluşur ama** pod'un macvlan arayüzü ya hiç trafik göremez ya da beklenmedik şekilde davranır. **Her zaman `br-ex`'i master olarak kullanın.**

- **Macvlan'ın kendine has host-pod izolasyon kısıtı:** Node'un kendisi (host netns), üzerinde çalışan macvlan pod'una **ping atamaz** — bu da canlı test edildi (pod'un çalıştığı node'dan pod'un `net1` IP'sine ping `%100 kayıp` verdi). Bu, Linux macvlan sürücüsünün tasarımı gereği normal/beklenen davranıştır (parent arayüz, kendi child'larına doğrudan ulaşamaz), bug değildir.
- **IP çakışması riski:** IPAM olarak `whereabouts` yerine sabit/manuel IP (`"ipam": {"type": "static", ...}`) kullanılırsa, birden fazla node'da paralel pod'lar aynı IP'yi alabilir (her node kendi CNI IPAM state'ini tutar, cluster genelinde koordinasyon olmaz) — LAN'da IP çakışmasına yol açar. Bu yüzden macvlan + gerçek LAN kombinasyonunda **whereabouts zorunlu görün**.
- **Node'lar VM ise (VMware vb.) — promiscuous mode / MAC ayarları:** Her macvlan arayüzü kendi MAC adresiyle konuşur. Node'lar bir hypervisor üzerinde VM olarak çalışıyorsa, port group / vSwitch seviyesinde **"MAC Address Changes"** ve **"Forged Transmits"** (gerekirse "Promiscuous Mode") izinli olmalıdır; aksi halde pod'ların trafiği hypervisor tarafından düşürülür. Bu ayar OpenShift'in dışında, sanallaştırma katmanında yapılır. Node içinde `ip -d link show br-ex` → `promiscuity` değeri Linux/OVS tarafının hazır olduğunu gösterir. Bare-metal node'larda bu sorun yoktur.
- **NetworkPolicy macvlan'ı kapsamaz:** Kubernetes `NetworkPolicy` kaynakları sadece cluster'ın birincil (OVN) arayüzünü kapsar; macvlan (`net1`) üzerinden gelen/giden trafiği **kısıtlamaz**. Macvlan pod'ları için erişim kontrolü LAN/switch/firewall seviyesinde ele alınmalıdır.

---

## 8. Temizlik

```bash
oc delete pod macvlan-test macvlan-test-2 -n ocp-poc
oc delete net-attach-def macvlan-lan macvlan-lan-routed -n ocp-poc
```

---

## Özet Tablo

| Bileşen | Rolü |
|---|---|
| `NetworkAttachmentDefinition` | Ek ağı (hangi CNI plugin, hangi master interface, hangi IPAM) tanımlar |
| `k8s.v1.cni.cncf.io/networks` annotation | Pod'u belirli bir NAD'a bağlar |
| `master: br-ex` | OVNKubernetes node'larında macvlan'ın gerçek fiziksel NIC'e çıkabilmesi için doğru hedef |
| `mode: bridge` | Aynı NAD'a bağlı pod'ların birbiriyle de konuşabilmesini sağlar |
| `whereabouts` IPAM | Cluster genelinde koordineli IP dağıtımı — çakışmayı önler |
| `net1` | Pod içinde macvlan arayüzünün göründüğü isim (`eth0` = cluster/OVN, `net1` = macvlan/LAN) |

**Altın kurallar:**
1. OVNKubernetes cluster'larında macvlan master'ı **fiziksel NIC değil, `br-ex`** olmalı.
2. Gerçek LAN'a çıkan macvlan demolarında IPAM olarak **her zaman whereabouts** kullanın, statik IP'yi elle dağıtmayın.
3. Pod'dan node'un kendisine macvlan üzerinden ping atamamak bir bug değil, kernel'in macvlan tasarımının doğal sonucudur.
4. NetworkPolicy macvlan trafiğini kapsamaz — erişim kontrolünü LAN seviyesinde planlayın.
5. **Farklı bir subnetten pod'a inbound erişim, `rp_filter=1` yüzünden varsayılan olarak çalışmaz.** Çözüm, NAD'ın IPAM'ine o subnet(ler) için `net1` gateway'i üzerinden **route** eklemektir (`macvlan-nad-routed.yaml`); node değişikliği veya reboot gerekmez.
6. Pod IP bloğu **DHCP havuzu dışında** olmalı ve kullanılmadan önce `arping` ile kontrol edilmelidir.
