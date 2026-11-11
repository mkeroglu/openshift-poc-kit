# 08 — Kubernetes / OpenShift NetworkPolicy Demo Rehberi

> 🇬🇧 [English](README.md) · [← 07 — Canary Deployment](../../B-application-delivery/07-canary-deployment/README.tr.md) · [POC akışı](../../README.tr.md) · [09 — Mikro-Segmentasyon →](../09-micro-segmentation/README.tr.md)

Bu doküman, canlı ortamda (pod içinde, `oc` ile) NetworkPolicy anlatımı yapmak için hazırlanmıştır. Tüm komutlar ve YAML'lar lab ortamında (OpenShift 4.22, OVNKubernetes) **uçtan uca test edilmiş**, çıktılar aşağıda gösterildiği gibi doğrulanmıştır. YAML'lar bu klasörde ayrı dosyalar olarak da bulunur. 4 namespace kullanılıyor: `poc-np-frontend`, `poc-np-backend`, `poc-np-other`, `poc-np-monitoring`.

Senaryo sırası:

1. NetworkPolicy nedir, neden gerekli
2. Ön koşul kontrolü (CNI/SDN desteği)
3. Demo ortamının kurulması (4 namespace + backend servis + test pod'ları)
4. Senaryo 1 — Bir namespace'teki tüm ingress trafiğini engelleme (deny-all)
5. Senaryo 2 — Sadece belirli bir namespace'den gelen trafiğe izin verme (+ ikinci namespace ekleyerek additive davranışı gösterme)
6. Senaryo 3 — Sadece belirli bir pod'dan / label'dan gelen trafiğe izin verme
7. Senaryo 4 — Egress (dışa çıkış) trafiğini kısıtlama + iki gerçek tuzak
8. Doğrulama / debug ipuçları
9. Temizlik

---

## 1. NetworkPolicy Nedir?

- NetworkPolicy, pod'lar arasındaki (ve pod ile dış dünya arasındaki) ağ trafiğini **hangi pod'ların, hangi namespace'lerin, hangi portların** birbirine erişebileceğini tanımlayan Kubernetes/OpenShift kaynağıdır.
- **Varsayılan davranış:** Bir namespace'te hiç NetworkPolicy yoksa, tüm pod'lar birbirine (ve dışarıya) serbestçe erişebilir (allow-all).
- Bir pod'u **seçen** (`podSelector` ile) en az bir NetworkPolicy oluşturulduğu an, o pod için **sadece o policy'lerin izin verdiği trafik** geçer — geri kalan her şey **default deny** olur. Yani "kısıtlama eklemek" aslında "izin listesi oluşturmak" demektir.
- Aynı namespace'te birden fazla NetworkPolicy varsa **additive/union (OR)** olarak birleşir — hiçbiri diğerini daraltmaz, sadece yeni izin ekler.
- İki yön vardır: `Ingress` (pod'a gelen trafik) ve `Egress` (pod'dan çıkan trafik).

> **Önemli:** NetworkPolicy'nin çalışması için cluster'ın SDN/CNI eklentisinin bunu desteklemesi gerekir. OpenShift 4.x'te varsayılan **OVNKubernetes** bunu tam destekler. Demo öncesi kontrol edin:

```bash
oc get network.config/cluster -o jsonpath='{.status.networkType}'
# Beklenen çıktı: OVNKubernetes
```

---

## 2. Demo Ortamının Kurulması

4 namespace (OpenShift'te "project") oluşturup her birine bir `team` label'ı veriyoruz — `namespaceSelector` bunu kullanacak.

```bash
for ns in poc-np-frontend poc-np-backend poc-np-other poc-np-monitoring; do
  oc new-project "$ns"
done

oc label namespace poc-np-frontend   team=frontend   --overwrite
oc label namespace poc-np-backend    team=backend    --overwrite
oc label namespace poc-np-other      team=other      --overwrite
oc label namespace poc-np-monitoring team=monitoring --overwrite
```

`poc-np-backend` içine hedef servisi kuruyoruz. **Not:** standart `nginx` image'ı OpenShift'in restricted SCC'si (rastgele UID) ile `/var/cache/nginx` yazma izni olmadığından çöker; s2i builder image'ları (`ubi9/nginx-124`) da doğrudan çalıştırılabilir bir sunucu değildir, sadece talimat yazdırır. Demo/test amaçlı en sorunsuz yol, arbitrary UID ile sorunsuz çalışan basit bir `busybox httpd`:

```bash
oc -n poc-np-backend create deployment web --image=busybox:1.36 \
  -- sh -c "mkdir -p /tmp/www && echo 'backend-ok' > /tmp/www/index.html && httpd -f -p 8080 -h /tmp/www"

oc -n poc-np-backend expose deployment web --port=8080

oc -n poc-np-backend get pods,svc
```

Her test namespace'ine bir "client" pod'u açıyoruz:

```bash
for ns in poc-np-frontend poc-np-other poc-np-monitoring; do
  oc -n $ns run tester --image=busybox:1.36 --restart=Never -- sleep 3600
done
```

**Baseline test (policy yokken hepsi başarılı olmalı):**

```bash
for ns in poc-np-frontend poc-np-other poc-np-monitoring; do
  echo "--- $ns -> backend ---"
  oc -n $ns exec tester -- wget -qO- --timeout=3 web.poc-np-backend.svc.cluster.local:8080
done
```

✅ **Gerçek test çıktısı — üçü de erişti:**

```
--- poc-np-frontend -> backend ---
backend-ok
--- poc-np-other -> backend ---
backend-ok
--- poc-np-monitoring -> backend ---
backend-ok
```

---

## 3. Senaryo 1: Tüm Trafiği Engelle (Default Deny)

`poc-np-backend` namespace'indeki tüm pod'lara gelen tüm ingress trafiğini engelliyoruz.

```yaml
# deny-all-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: poc-np-backend
spec:
  podSelector: {}      # namespace'teki TÜM pod'ları seçer
  policyTypes:
    - Ingress
  # ingress kuralı tanımlanmadığı için hiçbir trafiğe izin verilmez
```

```bash
oc apply -f deny-all-ingress.yaml
```

**Test — üçü de başarısız olmalı:**

```bash
for ns in poc-np-frontend poc-np-other poc-np-monitoring; do
  echo "--- $ns -> backend ---"
  oc -n $ns exec tester -- wget -qO- --timeout=3 web.poc-np-backend.svc.cluster.local:8080
done
```

✅ **Gerçek test çıktısı:**

```
--- poc-np-frontend -> backend ---
wget: download timed out
--- poc-np-other -> backend ---
wget: download timed out
--- poc-np-monitoring -> backend ---
wget: download timed out
```

Vurgulanacak nokta: `podSelector: {}` + boş `ingress` alanı → o namespace'teki tüm pod'lar için **her yönden** gelen trafik reddedilir.

---

## 4. Senaryo 2: Sadece Belirli Bir Namespace'den Gelen Trafiğe İzin Ver

Deny-all'ın üzerine, sadece `frontend` namespace'inden gelen trafiğe izin veren bir policy ekliyoruz.

```yaml
# allow-from-frontend-namespace.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-frontend-namespace
  namespace: poc-np-backend
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              team: frontend
      ports:
        - protocol: TCP
          port: 8080
```

```bash
oc apply -f allow-from-frontend-namespace.yaml
```

✅ **Gerçek test çıktısı:**

```
--- poc-np-frontend -> backend ---
backend-ok
--- poc-np-other -> backend ---
wget: download timed out
--- poc-np-monitoring -> backend ---
wget: download timed out
```

> **Not:** `namespaceSelector` etiket bazlıdır, isim bazlı değildir. OpenShift her namespace'e otomatik olarak `kubernetes.io/metadata.name` label'ını ekler, isimle eşleştirmek için de kullanılabilir:
> ```yaml
> - namespaceSelector:
>     matchLabels:
>       kubernetes.io/metadata.name: poc-np-frontend
> ```

### 4.1 Additive davranışı canlı göstermek: 4. namespace'i de ekleyelim

Aynı namespace'e **ikinci bir** allow policy ekleyerek `monitoring`'e de izin veriyoruz — mevcut policy'yi değiştirmeden:

```yaml
# allow-from-monitoring-namespace.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-monitoring-namespace
  namespace: poc-np-backend
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              team: monitoring
      ports:
        - protocol: TCP
          port: 8080
```

```bash
oc apply -f allow-from-monitoring-namespace.yaml
oc -n poc-np-backend get networkpolicy
```

✅ **Gerçek test çıktısı (4 namespace'in son durumu):**

```
--- poc-np-frontend -> backend ---
backend-ok
--- poc-np-other -> backend ---
wget: download timed out
--- poc-np-monitoring -> backend ---
backend-ok
```

```
NAME                              POD-SELECTOR   AGE
allow-from-frontend-namespace     <none>         ...
allow-from-monitoring-namespace   <none>         ...
default-deny-ingress              <none>         ...
```

Bu, "NetworkPolicy'ler birbirini ezmez, birleşir (OR)" mesajını canlı olarak kanıtlayan en net an: `frontend` ve `monitoring` giriyor, `other` hâlâ dışarıda.

---

## 5. Senaryo 3: Sadece Belirli Bir Pod'dan (Label) Gelen Trafiğe İzin Ver

Namespace bazlı izin bazen çok geniştir. Aynı namespace içinde bile sadece belirli label'lı pod'lardan izin vermek için `podSelector` kullanılır. `namespaceSelector` ve `podSelector` **aynı** `from` maddesi içinde birlikte kullanıldığında **AND** mantığıyla çalışır.

Önce `frontend` namespace'ine verilen geniş izni kaldırıp, yerine sadece belirli label'lı pod'a izin veren daha dar bir policy koyuyoruz:

```bash
oc -n poc-np-backend delete networkpolicy allow-from-frontend-namespace
```

```yaml
# allow-from-frontend-app-only.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-frontend-app-only
  namespace: poc-np-backend
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              team: frontend
          podSelector:
            matchLabels:
              role: api-client   # frontend namespace İÇİNDE sadece bu label'lı pod'lar
      ports:
        - protocol: TCP
          port: 8080
```

```bash
oc apply -f allow-from-frontend-app-only.yaml
```

**Test — label yokken erişim reddedilmeli:**

```bash
oc -n poc-np-frontend exec tester -- wget -qO- --timeout=3 web.poc-np-backend.svc.cluster.local:8080
```

✅ `wget: download timed out` (beklenen)

**Şimdi pod'a doğru label'ı ekleyip tekrar test edelim:**

```bash
oc -n poc-np-frontend label pod tester role=api-client
oc -n poc-np-frontend exec tester -- wget -qO- --timeout=3 web.poc-np-backend.svc.cluster.local:8080
```

✅ **Gerçek test çıktısı:** `backend-ok` — label eklenir eklenmez anında erişim açıldı (canlı demo için çok etkileyici bir an).

> **Dikkat (sık yapılan hata):** `from` altında iki ayrı liste elemanı yazılırsa (`- namespaceSelector...` ve ayrı satırda `- podSelector...`) bu **OR** anlamına gelir. AND için ikisi **aynı** liste elemanı içinde (yukarıdaki gibi, aynı `-` altında) olmalı.

---

## 6. Senaryo 4: Egress Trafiğini Kısıtlama (+ 1 gerçek tuzak)

`frontend` namespace'indeki pod'ların sadece `backend`'e (port 8080) ve DNS'e çıkış yapmasına izin verip başka her şeyi (örn. internet) engelliyoruz.

> **Düzeltme notu:** Bu bölümün ilk taslağında `to: []` (boş hedef listesi) yazmanın "hiçbir hedefe izin verme" anlamına geldiği iddia edilmişti. Bu **yanlıştı** — resmi API dokümantasyonuyla (`oc explain networkpolicy.spec.egress.to`) ve izole bir testle (yanlış DNS portunu devreden çıkarıp sadece `to: []` + doğru port ile IP üzerinden ham TCP handshake denemesi) doğrulandığı üzere, **`to` alanı boş veya hiç yazılmamışsa bu, "her hedefe izin ver" anlamına gelir** (sadece port'a göre kısıtlar, hedefe göre kısıtlamaz). İlk testte DNS'in kırılmasının gerçek sebebi aşağıdaki tek gerçek tuzaktı: yanlış port numarası.

### Gerçek tuzak: Service portu ≠ Pod portu

DNS namespace'i OpenShift'te `openshift-dns`, servis 53 numaralı portta dinliyor gibi görünür — ama gerçek pod (CoreDNS) **5353** portunu dinler, Service sadece 53→5353 DNAT'ı yapar. **NetworkPolicy'nin `ports` alanı servisin değil, hedef pod'un gerçek (target/container) portunu eşleştirir.** `port: 53` yazarsanız NetworkPolicy hiçbir zaman gerçek pod'la eşleşmez ve DNS sessizce kırılır.

Gerçek portu böyle doğruladık (`v1 Endpoints` 4.22'de deprecated uyarısı verir; EndpointSlice tercih edilir):

```bash
oc get endpointslice -n openshift-dns -l kubernetes.io/service-name=dns-default -o jsonpath='{.items[0].ports}'
# ✅ Gerçek çıktı: dns (UDP) 5353, dns-tcp (TCP) 5353, metrics (TCP) 9154
```

✅ **Gerçek çıktı — aynı policy `port: 53` ile yazıldığında:**

```
nslookup web.poc-np-backend.svc.cluster.local   ->  ;; connection timed out; no servers could be reached
wget web.poc-np-backend.svc.cluster.local:8080  ->  wget: bad address 'web.poc-np-backend.svc.cluster.local:8080'
```

Backend'e izin verildiği halde uygulama DNS çözemediği için sessizce kırılır.

### Doğru (test edilmiş, çalışan) egress policy'si

```yaml
# restrict-egress-frontend.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-egress
  namespace: poc-np-frontend
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              team: backend
      ports:
        - protocol: TCP
          port: 8080
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: openshift-dns
      ports:
        - protocol: UDP
          port: 5353
        - protocol: TCP
          port: 5353
```

```bash
oc apply -f restrict-egress-frontend.yaml
```

✅ **Gerçek test çıktısı:**

```bash
oc -n poc-np-frontend exec tester -- nslookup web.poc-np-backend.svc.cluster.local
# Server:  172.30.0.10   Address: 172.30.0.10:53
# Name:    web.poc-np-backend.svc.cluster.local
# Address: 172.30.85.177          <-- DNS çözümü çalışıyor

oc -n poc-np-frontend exec tester -- wget -qO- --timeout=3 web.poc-np-backend.svc.cluster.local:8080
# backend-ok                       <-- backend erişimi çalışıyor

oc -n poc-np-frontend exec tester -- wget -qO- --timeout=3 http://1.1.1.1
# timeout                          <-- internet'e çıkış engellendi (beklenen)
```

> **Demo notu:** Bu tuzağı (service-port/pod-port farkı) canlı gösterip düzeltmek, "gerçekten neyi test ettiğinizi bilmiyorsanız NetworkPolicy sizi sessizce kırar" mesajını güçlü şekilde iletir. Kubernetes clusterlarında genelde `kube-dns` port 53 = pod port 53 olduğu için bu tuzak çıkmaz; OpenShift'e özgü bir detaydır — vanilla Kubernetes'te farklı davranabileceğini belirtin.
>
> **`to`/`from` alanı hakkında doğru bilgi:** `to: []`, boş bırakmak veya alanı hiç yazmamak — üçü de aynı anlama gelir: **"hedef/kaynak bakımından kısıtlama yok, sadece port'a göre filtrele."** Kısıtlama istiyorsanız listeye en az bir `namespaceSelector`/`podSelector`/`ipBlock` eklemeniz gerekir (bu doğruluğu `oc explain networkpolicy.spec.egress.to` çıktısıyla ve canlı bir izole testle doğruladık).

---

## 7. Doğrulama / Debug İpuçları

```bash
# Namespace'teki tüm policy'leri listele
oc get networkpolicy -n poc-np-backend

# Detay ve seçici bilgisi
oc describe networkpolicy default-deny-ingress -n poc-np-backend

# Cluster'ın SDN tipini doğrula (OVNKubernetes NetworkPolicy'yi destekler)
oc get network.config/cluster -o jsonpath='{.status.networkType}'

# Bir servisin GERÇEK hedef portunu bulmak (egress policy yazarken kritik)
oc get endpointslice -n <namespace> -l kubernetes.io/service-name=<servis-adi> -o jsonpath='{.items[0].ports}'

# Namespace label'larını kontrol etmek (namespaceSelector hataları için)
oc get ns <namespace> --show-labels
```

---

## 8. Temizlik

```bash
oc delete project poc-np-frontend poc-np-backend poc-np-other poc-np-monitoring
```

---

## Özet Tablo

| Senaryo | policyTypes | Ana alan |
|---|---|---|
| Tüm trafiği engelle | Ingress | `podSelector: {}`, boş `ingress` |
| Belirli namespace'e izin ver | Ingress | `ingress.from.namespaceSelector` |
| İkinci namespace'i additive ekle | Ingress | Aynı ns'e 2. bir NetworkPolicy objesi (OR) |
| Belirli pod'a izin ver | Ingress | `ingress.from.podSelector` (+ namespaceSelector = AND, aynı liste elemanında) |
| Egress kısıtlama | Egress | `egress.to` — kısıtlamak için içine en az bir selector eklenmeli |

**Altın kurallar:**
1. Bir pod'u seçen ilk NetworkPolicy oluşturulduğu anda o yön (ingress/egress) için default-deny devreye girer; sonrasında eklenen her policy sadece **ek izin** (allow/OR) tanımlar, asla kısıtlama sıkılaştırmaz.
2. `to`/`from` alanını boş dizi (`[]`) bırakmak **da**, alanı tamamen atlamak **da** aynı anlama gelir: **"hedef/kaynak bakımından kısıtlama yok"** (sadece `ports` filtre olarak kalır). Hedefi/kaynağı kısıtlamak için listeye en az bir `namespaceSelector`/`podSelector`/`ipBlock` yazmak gerekir. (Kaynak: `oc explain networkpolicy.spec.egress.to`, canlı izole testle doğrulandı.)
3. `ports` alanı her zaman **hedef pod'un gerçek container portunu** eşleştirir, Service'in dışarı sunduğu portu değil — DNS gibi port-mapping yapan servislerde `oc get endpoints` ile gerçek portu doğrulayın.
