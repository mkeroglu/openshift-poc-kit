# 06 — Blue/Green Deployment Demo Rehberi

> 🇬🇧 [English](README.md) · [← 05 — CI/CD](../05-ci-cd/README.tr.md) · [POC akışı](../../README.tr.md) · [07 — Canary Deployment →](../07-canary-deployment/README.tr.md)

Bu doküman, OpenShift üzerinde **Blue/Green deployment** stratejisini anlatır. Bu, [05 — CI/CD](../05-ci-cd/README.tr.md) bölümündeki pipeline otomasyonunun bir devamı/alternatifi olarak düşünülebilir — orada `oc rollout restart` ile tek bir Deployment'ı yerinde (rolling) güncelliyorduk, burada **iki bağımsız ortamı** aynı anda ayakta tutup aralarında anlık geçiş yapıyoruz. Tüm adımlar lab ortamında (OpenShift 4.22) **`poc-bg-demo` namespace'inde canlı test edilmiştir**. Manifestlerin tamamı `blue-green-demo.yaml` dosyasındadır.

Senaryo sırası:

1. Kavram
2. Uygulama: iki Deployment + tek Service + Route
3. Doğrulama (gerçek test çıktıları — switch ve rollback)
4. Gerçek tuzak: router gecikmesi
5. Temizlik

---

## 1. Kavram

Blue/Green deployment'ta **iki ayrı, tamamen bağımsız** ortam (Deployment) aynı anda çalışır durumda tutulur — biri o an trafiği alan (**örn. Blue**), diğeri yeni versiyonu çalıştıran ama henüz trafik almayan (**örn. Green**). Geçiş, tek bir **Service selector**'ının (veya Route hedefinin) Blue'dan Green'e çevrilmesiyle **anlık ve tek adımda** yapılır — Rolling update'in aksine, eski ve yeni pod'lar arada karışık trafik almaz, geçiş net bir "anahtar" gibi işler. Aynı mekanizma tersine çevrilerek **anında rollback** için de kullanılır (Green'de sorun çıkarsa selector'ı tekrar Blue'ya çevirmek yeterli, yeniden deploy gerekmez).

[05 — CI/CD](../05-ci-cd/README.tr.md)'deki `pipeline-build-deploy.yaml`'ın kullandığı `oc rollout restart` (tek Deployment, rolling update) modelinden farklı olarak, Blue/Green **iki ayrı Deployment** ve bunları birleştiren **tek bir Service** gerektirir.

---

## 2. Uygulama: İki Deployment + Tek Service + Route

```yaml
# blue-green-demo.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: poc-bg-demo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-blue
  namespace: poc-bg-demo
  labels:
    app: demo-app
    version: blue
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo-app
      version: blue
  template:
    metadata:
      labels:
        app: demo-app
        version: blue
    spec:
      containers:
        - name: web
          image: busybox:1.36
          command: ["sh", "-c", "mkdir -p /tmp/www && echo BLUE > /tmp/www/index.html && httpd -f -p 8080 -h /tmp/www"]
          ports:
            - containerPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-green
  namespace: poc-bg-demo
  labels:
    app: demo-app
    version: green
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo-app
      version: green
  template:
    metadata:
      labels:
        app: demo-app
        version: green
    spec:
      containers:
        - name: web
          image: busybox:1.36
          command: ["sh", "-c", "mkdir -p /tmp/www && echo GREEN > /tmp/www/index.html && httpd -f -p 8080 -h /tmp/www"]
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: demo-app
  namespace: poc-bg-demo
spec:
  selector:
    app: demo-app
    version: blue        # <-- trafiği yönlendiren TEK alan; Blue/Green geçişi bunu değiştirmektir
  ports:
    - port: 8080
      targetPort: 8080
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: demo-app
  namespace: poc-bg-demo
spec:
  to:
    kind: Service
    name: demo-app
  port:
    targetPort: 8080
```

```bash
oc apply -f blue-green-demo.yaml
oc get deployment -n poc-bg-demo
```

✅ **Gerçek çıktı:** `app-blue` ve `app-green` her ikisi de `2/2 READY` — iki ortam **aynı anda** ayakta, sadece Service'in seçtiği hangisiyse o trafik alıyor.

---

## 3. Doğrulama (gerçek test çıktıları)

**a) Başlangıç durumu — Service Blue'yu seçiyor:**

```bash
HOST=$(oc get route demo-app -n poc-bg-demo -o jsonpath='{.spec.host}')
curl -s "http://$HOST"
```

✅ `BLUE`

**b) Switch — Service selector'ını Green'e çevir:**

```bash
oc patch service demo-app -n poc-bg-demo --type merge \
  -p '{"spec":{"selector":{"app":"demo-app","version":"green"}}}'
```

```bash
for i in 1 2 3 4 5; do curl -s "http://$HOST"; echo; done
```

✅ **Gerçek çıktı:** Switch sonrasında 0,1–0,2 sn aralıkla gönderilen isteklerle ölçülen, ilk `GREEN` cevabına kadar geçen süre (4 tur):

| Tur | Blue → Green | Green → Blue (rollback) | Hatalı/boş cevap |
|---|---|---|---|
| 1 | 1,67 sn | 2,36 sn | 0 |
| 2 | 0,25 sn | 0,14 sn | 0 |
| 3 | 0,25 sn | 0,14 sn | 0 |
| 4 | 0,25 sn | 4,02 sn | 0 |

Geçiş sırasında **hiçbir istek hata vermedi**; geçiş anına kadar eski sürüm, sonrasında yeni sürüm cevap verdi. Süre ise sabit değil (bkz. Bölüm 4).

**c) Rollback — anında Blue'ya geri dön:**

```bash
oc patch service demo-app -n poc-bg-demo --type merge \
  -p '{"spec":{"selector":{"app":"demo-app","version":"blue"}}}'
```

✅ **Gerçek çıktı:** Rollback da yeniden deploy/build gerekmeden, tek bir `oc patch` ile 0,14–4 sn içinde tamamlandı (yukarıdaki tablo).

---

## 4. Gerçek Tuzak: Router Gecikmesi

Switch anında değil — OpenShift router'ının (HAProxy) yeni `Endpoints`'i alıp backend havuzunu güncellemesi **anlık değil** ve **değişken**: canlı testlerde 0,14 sn ile 4 sn arasında ölçüldü (önceki bir test turunda ~5 sn de görüldü). Kök neden bug değil, router'ın endpoint değişikliklerini yakalayıp HAProxy config'ini reload etmesinin doğal (küçük) gecikmesi.

**Pratik sonuç:** Blue/Green switch'ini "tamamlandı" saymadan önce, gerçekten tüm trafiğin yeni versiyona geçtiğini birkaç saniye arayla tekrar test ederek doğrulayın — özellikle otomasyonda (CI/CD script'i) switch sonrası sabit bir bekleme payı (`sleep`) bırakıp ardından doğrulama yapın, "patch komutu döndü = trafik geçti" varsayımıyla hemen ilerlemeyin.

---

## 5. Temizlik

```bash
oc delete namespace poc-bg-demo
```

---

## Özet Tablo

| Bileşen | Rolü |
|---|---|
| `app-blue` / `app-green` Deployment | İki bağımsız, aynı anda ayakta duran ortam |
| `Service.spec.selector` | Trafiği yönlendiren tek anahtar — `version: blue` ↔ `version: green` |
| `Route` | Dış erişim, Service'i hedefler, switch sırasında değişmez |

**Altın kurallar:**
1. Blue/Green, Rolling update'ten farklı olarak **iki kat kaynak** (CPU/memory) gerektirir — her iki ortam da aynı anda ayakta tutulur.
2. Switch = tek bir `Service.spec.selector` değişikliği; rollback da aynı mekanizma, ters yönde.
3. Switch sonrası router'ın endpoint güncellemesi **anlık değil** — birkaç saniyelik gecikmeyi hesaba katıp doğrulama yapın, hemen "tamamlandı" varsaymayın.
