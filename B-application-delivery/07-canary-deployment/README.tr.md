# 07 — Canary Deployment Demo Rehberi

> 🇬🇧 [English](README.md) · [← 06 — Blue/Green Deployment](../06-blue-green-deployment/README.tr.md) · [POC akışı](../../README.tr.md) · [08 — NetworkPolicy →](../../C-network-and-security/08-network-policy/README.tr.md)

Bu doküman, OpenShift üzerinde **Canary deployment** stratejisini anlatır. [06 — Blue/Green Deployment](../06-blue-green-deployment/README.tr.md)'dan farkı: trafik tek seferde %100 geçmez, **kademeli bir yüzdeyle** (`%10 → %50 → %100`) yeni versiyona kaydırılır — amaç, yeni versiyonda bir sorun varsa kullanıcıların yalnızca küçük bir kısmının etkilenmesi ve geri dönüşün (yüzdeyi düşürmenin) kolay olmasıdır. Ek bir bileşen (Service Mesh vb.) gerekmeden, doğrudan **OpenShift Route**'un `alternateBackends` + `weight` alanlarıyla, HAProxy router'ın kendi trafik bölme mekanizması kullanılarak yapılmıştır. Tüm adımlar lab ortamında (OpenShift 4.22) **`poc-canary-demo` namespace'inde canlı test edilmiştir** — her aşamada 100–200 gerçek istek atılıp dağılım gerçekten ölçülmüştür. Manifestlerin tamamı `canary-demo.yaml` dosyasındadır.

Senaryo sırası:

1. Kavram
2. Uygulama: iki Deployment + iki Service + ağırlıklı (weighted) Route
3. Doğrulama (gerçek ölçülmüş dağılımlar — %90/10, %50/50, %100/0, rollback)
4. Notlar
5. Temizlik

---

## 1. Kavram

- **Blue/Green**'de trafik `Service.spec.selector` değiştirilerek **tek adımda %100** geçer.
- **Canary**'de trafik, **aynı anda ayakta olan iki farklı Service**'e (stable + canary) bir **Route** üzerinden **ağırlık (weight) oranına göre dağıtılır**. OpenShift Route'ta bu, `spec.to.weight` (birincil backend) ve `spec.alternateBackends[].weight` (ikincil backend'ler) alanlarıyla tanımlanır — router, gelen her isteği bu ağırlıkların oranına göre olasılıksal olarak (round-robin/random, ağırlıklı) iki Service arasında paylaştırır.
- Tipik akış: küçük bir yüzdeyle başla (**%10**) → metrik/hata oranını izle → sorun yoksa yüzdeyi kademeli artır (**%50**) → sonunda **%100**'e (tam promotion) geç. Herhangi bir aşamada sorun görülürse, yüzdeyi anında **0**'a çekmek yeterli — rollback için yeniden deploy gerekmez.

---

## 2. Uygulama: İki Deployment + İki Service + Ağırlıklı Route

```yaml
# canary-demo.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: poc-canary-demo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-stable
  namespace: poc-canary-demo
  labels:
    app: demo-app
    version: stable
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo-app
      version: stable
  template:
    metadata:
      labels:
        app: demo-app
        version: stable
    spec:
      containers:
        - name: web
          image: busybox:1.36
          command: ["sh", "-c", "mkdir -p /tmp/www && echo STABLE > /tmp/www/index.html && httpd -f -p 8080 -h /tmp/www"]
          ports:
            - containerPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-canary
  namespace: poc-canary-demo
  labels:
    app: demo-app
    version: canary
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo-app
      version: canary
  template:
    metadata:
      labels:
        app: demo-app
        version: canary
    spec:
      containers:
        - name: web
          image: busybox:1.36
          command: ["sh", "-c", "mkdir -p /tmp/www && echo CANARY > /tmp/www/index.html && httpd -f -p 8080 -h /tmp/www"]
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: app-stable
  namespace: poc-canary-demo
spec:
  selector:
    app: demo-app
    version: stable
  ports:
    - port: 8080
      targetPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: app-canary
  namespace: poc-canary-demo
spec:
  selector:
    app: demo-app
    version: canary
  ports:
    - port: 8080
      targetPort: 8080
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: demo-app
  namespace: poc-canary-demo
spec:
  to:
    kind: Service
    name: app-stable
    weight: 90              # <-- ana trafiğin %90'ı stable'a
  alternateBackends:
    - kind: Service
      name: app-canary
      weight: 10             # <-- %10'u canary'ye
  port:
    targetPort: 8080
```

```bash
oc apply -f canary-demo.yaml
oc get deployment -n poc-canary-demo
```

✅ **Gerçek çıktı:** `app-stable` ve `app-canary` her ikisi de `2/2 READY` — Blue/Green'deki gibi iki ortam aynı anda ayakta, farkı **tek bir Service**'e değil **iki ayrı Service**'e (Route üzerinden ağırlıklı) trafik gitmesi.

---

## 3. Doğrulama (gerçek ölçülmüş dağılımlar)

Her aşamada aynı test: **100 gerçek `curl` isteği** atılıp `STABLE`/`CANARY` cevaplarının sayısı tutuldu.

```bash
HOST=$(oc get route demo-app -n poc-canary-demo -o jsonpath='{.spec.host}')
STABLE=0; CANARY=0
for i in $(seq 1 100); do
  R=$(curl -s "http://$HOST")
  if [ "$R" = "STABLE" ]; then STABLE=$((STABLE+1)); elif [ "$R" = "CANARY" ]; then CANARY=$((CANARY+1)); fi
done
echo "STABLE: $STABLE  CANARY: $CANARY"
```

### a) Başlangıç: `%90 stable / %10 canary`

✅ **Gerçek çıktı (200 istek):** `STABLE: 181  CANARY: 19` (%90,5 / %9,5) — yapılandırılan oranla uyumlu. Önceki bir test turunda 100 istekte `91 / 9` ölçülmüştü.

### b) Yükselt: `%50 / %50`

```bash
oc patch route demo-app -n poc-canary-demo --type=json -p '[
  {"op":"replace","path":"/spec/to/weight","value":50},
  {"op":"replace","path":"/spec/alternateBackends/0/weight","value":50}
]'
```

✅ **Gerçek çıktı:** `STABLE: 50  CANARY: 50` — tam ortadan.

### c) Tam promotion: `%0 stable / %100 canary`

```bash
oc patch route demo-app -n poc-canary-demo --type=json -p '[
  {"op":"replace","path":"/spec/to/weight","value":0},
  {"op":"replace","path":"/spec/alternateBackends/0/weight","value":100}
]'
```

✅ **Gerçek çıktı (100 istek):** `STABLE: 0  CANARY: 100` — `weight: 0` verildiğinde stable'a **hiç** istek gitmiyor, tam geçiş doğrulandı.

### d) Rollback: anında `%100 stable / %0 canary`

```bash
oc patch route demo-app -n poc-canary-demo --type=json -p '[
  {"op":"replace","path":"/spec/to/weight","value":100},
  {"op":"replace","path":"/spec/alternateBackends/0/weight","value":0}
]'
```

✅ **Gerçek çıktı (100 istek):** `STABLE: 100  CANARY: 0` — tek bir `oc patch` ile, yeniden deploy gerekmeden anında eski versiyona dönüldü. Tüm aşamalarda hatalı/boş cevap sayısı **0**.

---

## 4. Notlar

- **Blue/Green'deki router gecikmesiyle karşılaştırma:** Blue/Green demosunda (`Service.spec.selector` değişikliği) switch'in router'a yansıması 0,1–4 sn arasında değişiyordu. Canary'de (`Route.spec` ağırlık değişikliği) her değişiklikten sonra 3 sn beklenip ölçüm yapıldı; ilk istekten itibaren yeni oran tutarlıydı. Yine de otomasyonda her ağırlık değişikliğinden sonra kısa bir bekleme payı bırakıp öyle ölçüm/karar almak güvenli bir alışkanlıktır.
- **Ağırlıklar oransal**, mutlak yüzde değil: `weight: 90` / `weight: 10` ile `weight: 9` / `weight: 1` matematiksel olarak aynı %90/%10 oranını verir — router ağırlıkları toplamına göre normalize eder.
- **Sticky session (oturum yapışkanlığı):** OpenShift router, HTTP/edge route'larda **varsayılan olarak** bir sticky cookie verir (`set-cookie: <hash>=...; HttpOnly`). Cookie'yi saklayan istemciler (tarayıcılar) ilk istekte ağırlığa göre bir sürüme atanır ve **sonra hep aynı sürümde kalır**; bu, canary'de kullanıcının sürümler arasında gidip gelmemesi için istenen davranıştır. Cookie saklamayan istemciler (curl, bazı API istemcileri) her istekte yeniden dağıtılır.

  ✅ **Gerçek çıktı (%50/%50):**

  | İstemci | 20 isteğin sonucu |
  |---|---|
  | Cookie saklayan istemci 1, 2, 5 | 20/20 `CANARY` |
  | Cookie saklayan istemci 3, 4 | 20/20 `STABLE` |
  | Cookie saklamayan istemci | 10 `STABLE` / 10 `CANARY` |
  | `haproxy.router.openshift.io/balance=source` + `disable_cookies=true` (tek istemci IP'si, 40 istek) | 40/40 `STABLE` |

  `balance=source` istemci IP'sine göre yapıştırır. Kullanıcılar NAT/proxy arkasındaysa çok sayıda kullanıcı aynı IP'den geleceği için dağılım bozulur; bu durumda cookie tercih edilmelidir.
- Daha gelişmiş canary senaryoları (otomatik metrik-bazlı ilerletme/geri alma, HTTP header/kullanıcı bazlı yönlendirme) için **OpenShift Service Mesh** (Istio) daha uygun bir araçtır; buradaki yöntem ek bileşen gerektirmeyen, native OpenShift Route mekanizmasıdır.

---

## 5. Temizlik

```bash
oc delete namespace poc-canary-demo
```

---

## Özet Tablo

| Bileşen | Rolü |
|---|---|
| `app-stable` / `app-canary` Deployment + Service | İki bağımsız ortam, **iki ayrı** Service (Blue/Green'de tek Service'ti) |
| `Route.spec.to.weight` | Birincil backend'in (stable) aldığı göreli trafik payı |
| `Route.spec.alternateBackends[].weight` | İkincil backend'in (canary) aldığı göreli trafik payı |

**Altın kurallar:**
1. Canary, Blue/Green'in kademeli hâli — aynı "iki ortamı aynı anda ayakta tutma" prensibi, ama trafik `Service.selector` yerine **Route ağırlıkları** ile bölünür.
2. Ağırlıklar **oransaldır**, mutlak değer değil — `weight: 0` bir backend'i tamamen devre dışı bırakır.
3. Her ağırlık değişikliğinden sonra **gerçek trafikle ölçüm yaparak** doğrulayın — "patch komutu başarılı döndü" ≠ "trafik istenen oranda dağılıyor".
4. Router varsayılanda cookie ile sticky çalışır: tarayıcı kullanıcıları bir sürüme yapışır, cookie saklamayan istemciler her istekte yeniden dağıtılır. Testlerde bunu hesaba katın.
