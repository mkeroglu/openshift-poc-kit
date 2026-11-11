# 09 — Mikro-Segmentasyon Demo Rehberi (Katman/Rol Bazlı İzolasyon)

> 🇬🇧 [English](README.md) · [← 08 — NetworkPolicy](../08-network-policy/README.tr.md) · [POC akışı](../../README.tr.md) · [10 — Multus CNI →](../10-multus-cni/README.tr.md)

Bu doküman, **aynı namespace içindeki** iş yüklerini **katman/rol** (`tier` label'ı) bazında izole edip **yalnızca izinli akışların** geçmesini sağlayan bir mikro-segmentasyon senaryosunu anlatır. [08 — NetworkPolicy](../08-network-policy/README.tr.md) rehberindeki demo **namespace bazlıydı** (`frontend`/`backend`/`monitoring` ayrı namespace'lerdi, `namespaceSelector` kullanılmıştı); burada asıl fark — **tek bir namespace içinde**, pod'ların `tier` label'ına göre (`podSelector`) izolasyon uygulanması, yani "namespace sınırı" değil **iş yükünün rolü** izolasyon birimi oluyor. Klasik 3 katmanlı mimari (**frontend → backend → database**) üzerinden, frontend'in backend'i **atlayarak** database'e doğrudan erişememesi canlı olarak kanıtlanmıştır.

Tüm adımlar lab ortamında (OpenShift 4.22, OVNKubernetes) **`poc-microseg-demo` namespace'inde canlı test edilmiştir**. YAML'lar bu klasörde ayrı dosyalar olarak da bulunur.

Senaryo sırası:

1. Kavram — neden "mikro"?
2. Demo ortamının kurulması (3 katman, tek namespace)
3. Baseline testi (policy yokken — hepsi birbirine erişebilir)
4. Mikro-segmentasyon NetworkPolicy'leri
5. Doğrulama (gerçek test çıktıları — kritik test: frontend'in database'i atlayamaması)
6. Temizlik

---

## 1. Kavram — Neden "Mikro"?

- **Makro-segmentasyon** (bkz. [08 — NetworkPolicy](../08-network-policy/README.tr.md) demosu): izolasyon birimi **namespace**'tir — "bu namespace'teki hiçbir pod, şu namespace'teki pod'lara erişemez" gibi kaba taneli kurallar.
- **Mikro-segmentasyon**: izolasyon birimi **tek tek pod/iş yükü rolüdür** — aynı namespace içinde bile, sadece **belirli bir rolün** (`tier: frontend`, `tier: backend`, `tier: database` gibi) başka **belirli bir role** erişimine izin verilir, geri kalan her şey (aynı namespace içinde olsa dahi) reddedilir.
- Pratik önemi: bir saldırgan/hatalı bir servis **frontend** katmanına sızsa bile, **backend'i atlayıp doğrudan database'e** erişememesi gerekir. Bu, "doğru namespace'e girdiysen her şeye erişebilirsin" varsayımını kırar — **zero-trust** yaklaşımının temelidir.

---

## 2. Demo Ortamının Kurulması

Tek bir namespace'te 3 katman (her biri `tier` label'ıyla ayrılmış):

```yaml
# tiers.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: poc-microseg-demo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: poc-microseg-demo
  labels:
    tier: frontend
spec:
  replicas: 1
  selector:
    matchLabels: { tier: frontend }
  template:
    metadata:
      labels: { tier: frontend }
    spec:
      containers:
        - name: web
          image: busybox:1.36
          command: ["sh", "-c", "mkdir -p /tmp/www && echo FRONTEND-OK > /tmp/www/index.html && httpd -f -p 8080 -h /tmp/www"]
          ports: [{ containerPort: 8080 }]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: poc-microseg-demo
  labels:
    tier: backend
spec:
  replicas: 1
  selector:
    matchLabels: { tier: backend }
  template:
    metadata:
      labels: { tier: backend }
    spec:
      containers:
        - name: web
          image: busybox:1.36
          command: ["sh", "-c", "mkdir -p /tmp/www && echo BACKEND-OK > /tmp/www/index.html && httpd -f -p 8080 -h /tmp/www"]
          ports: [{ containerPort: 8080 }]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
  namespace: poc-microseg-demo
  labels:
    tier: database
spec:
  replicas: 1
  selector:
    matchLabels: { tier: database }
  template:
    metadata:
      labels: { tier: database }
    spec:
      containers:
        - name: web
          image: busybox:1.36
          command: ["sh", "-c", "mkdir -p /tmp/www && echo DATABASE-OK > /tmp/www/index.html && httpd -f -p 8080 -h /tmp/www"]
          ports: [{ containerPort: 8080 }]
---
apiVersion: v1
kind: Service
metadata: { name: frontend, namespace: poc-microseg-demo }
spec: { selector: { tier: frontend }, ports: [{ port: 8080 }] }
---
apiVersion: v1
kind: Service
metadata: { name: backend, namespace: poc-microseg-demo }
spec: { selector: { tier: backend }, ports: [{ port: 8080 }] }
---
apiVersion: v1
kind: Service
metadata: { name: database, namespace: poc-microseg-demo }
spec: { selector: { tier: database }, ports: [{ port: 8080 }] }
```

```bash
oc apply -f tiers.yaml
oc get deployment -n poc-microseg-demo
```

✅ **Gerçek çıktı:** `frontend`, `backend`, `database` — üçü de `1/1 READY`.

---

## 3. Baseline Testi (Policy Yokken)

NetworkPolicy uygulamadan önce, varsayılan davranışın **allow-all** olduğunu doğrulayın:

```bash
FRONTEND_POD=$(oc get pod -n poc-microseg-demo -l tier=frontend -o jsonpath='{.items[0].metadata.name}')
BACKEND_POD=$(oc get pod -n poc-microseg-demo -l tier=backend -o jsonpath='{.items[0].metadata.name}')

oc exec "$FRONTEND_POD" -n poc-microseg-demo -- wget -qO- --timeout=3 backend.poc-microseg-demo.svc.cluster.local:8080
oc exec "$BACKEND_POD"  -n poc-microseg-demo -- wget -qO- --timeout=3 database.poc-microseg-demo.svc.cluster.local:8080
oc exec "$FRONTEND_POD" -n poc-microseg-demo -- wget -qO- --timeout=3 database.poc-microseg-demo.svc.cluster.local:8080
```

✅ **Gerçek çıktı:** üçü de başarılı (`BACKEND-OK`, `DATABASE-OK`, `DATABASE-OK`) — **frontend, policy yokken database'e de doğrudan erişebiliyor**. Bu, aşağıdaki NetworkPolicy'lerin çözeceği tam olarak bu sorundur.

---

## 4. Mikro-Segmentasyon NetworkPolicy'leri

Üç kural: **(1)** her şeyi varsayılan reddet, **(2)** sadece frontend→backend'e izin ver, **(3)** sadece backend→database'e izin ver. Hepsi **`podSelector`** kullanıyor (`namespaceSelector` değil) — izolasyon birimi rol, namespace değil.

```yaml
# microseg-policies.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: poc-microseg-demo
spec:
  podSelector: {}          # namespace'teki TÜM pod'lar
  policyTypes: [Ingress]
  # ingress kuralı yok -> hiçbir trafiğe izin verilmez (varsayılan red)
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: poc-microseg-demo
spec:
  podSelector:
    matchLabels: { tier: backend }     # bu kural SADECE backend pod'larını korur
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector:
            matchLabels: { tier: frontend }   # SADECE frontend'den
      ports:
        - protocol: TCP
          port: 8080
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-database
  namespace: poc-microseg-demo
spec:
  podSelector:
    matchLabels: { tier: database }    # bu kural SADECE database pod'larını korur
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector:
            matchLabels: { tier: backend }    # SADECE backend'den
      ports:
        - protocol: TCP
          port: 8080
```

```bash
oc apply -f microseg-policies.yaml
```

> ⚠️ **Dikkat — Route trafiği de kesilir:** `frontend` için bir "allow" kuralı yazılmadığında `default-deny-ingress`, **OpenShift router'ından (Route) gelen dış trafiği de** keser; uygulama kullanıcılara kapanır.
>
> ✅ **Gerçek çıktı:** Policy'ler öncesinde `curl http://<frontend-route>` → `200`; `microseg-policies.yaml` uygulandıktan sonra → `000` (zaman aşımı).

Çözüm: Sadece router'ın bulunduğu namespace'ten frontend'e gelen trafiğe izin veren bir kural (`allow-ingress-to-frontend.yaml`). OpenShift, ingress controller namespace'ini `policy-group.network.openshift.io/ingress: ""` label'ıyla işaretler:

```yaml
# allow-ingress-to-frontend.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-to-frontend
  namespace: poc-microseg-demo
spec:
  podSelector:
    matchLabels: { tier: frontend }
  policyTypes: [Ingress]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              policy-group.network.openshift.io/ingress: ""
      ports:
        - protocol: TCP
          port: 8080
```

```bash
oc expose svc frontend -n poc-microseg-demo
oc apply -f allow-ingress-to-frontend.yaml
```

✅ **Gerçek çıktı:** Route → `FRONTEND-OK [200]`. `database` için de bir Route açılsa bile dışarıdan erişim `000` kaldı; çünkü router'a sadece frontend için izin verildi.

---

## 5. Doğrulama (gerçek test çıktıları)

**Aynı 3 testi, policy'ler devredeyken tekrarlayın:**

```bash
echo "frontend -> backend:"
oc exec "$FRONTEND_POD" -n poc-microseg-demo -- wget -qO- --timeout=3 backend.poc-microseg-demo.svc.cluster.local:8080

echo "backend -> database:"
oc exec "$BACKEND_POD" -n poc-microseg-demo -- wget -qO- --timeout=3 database.poc-microseg-demo.svc.cluster.local:8080

echo "frontend -> database (DOGRUDAN, backend atlanarak):"
oc exec "$FRONTEND_POD" -n poc-microseg-demo -- wget -qO- --timeout=3 database.poc-microseg-demo.svc.cluster.local:8080
```

✅ **Gerçek çıktı:**

```
frontend -> backend:
BACKEND-OK                          <-- izinli akış, çalışıyor

backend -> database:
DATABASE-OK                         <-- izinli akış, çalışıyor

frontend -> database (DOGRUDAN, backend atlanarak):
wget: download timed out            <-- KRİTİK TEST: engellendi
command terminated with exit code 1
```

**Sonuç: frontend ve database aynı namespace'te, aynı cluster network'ünde olmalarına rağmen, aralarında `podSelector` bazlı bir kural olmadığı için iletişim tamamen kesildi.** Frontend, backend'i atlayarak database'e **hiçbir şekilde** ulaşamıyor — mikro-segmentasyonun temel vaadi budur.

> **Not:** `tier` label'ı taşımayan bir test pod'u (örn. genel bir "tester" pod'u) ile deneme yapmayın — `default-deny-ingress` onu da (haklı olarak) engeller, "her şey kırık" gibi yanıltıcı bir sonuç verir. Testleri her zaman **gerçek rol pod'larının içinden `oc exec`** ile yapın, testte kullanılan kimliğin de segmentasyona tabi olduğunu unutmayın.

---

## 6. Temizlik

```bash
oc delete project poc-microseg-demo
```

---

## Özet Tablo

| İzin verilen akış | Hangi NetworkPolicy | Test sonucu |
|---|---|---|
| frontend → backend :8080 | `allow-frontend-to-backend` | ✅ `BACKEND-OK` |
| backend → database :8080 | `allow-backend-to-database` | ✅ `DATABASE-OK` |
| frontend → database :8080 (doğrudan) | *(hiçbiri — kasıtlı olarak yok)* | ❌ timeout |
| Dış dünya (Route) → frontend | `allow-ingress-to-frontend` | ✅ `FRONTEND-OK` (kural yokken `000`) |
| Dış dünya (Route) → database | *(hiçbiri)* | ❌ `000` |

**Altın kurallar:**
1. Mikro-segmentasyonda izolasyon birimi **namespace değil, iş yükünün rolüdür** (`podSelector`, `namespaceSelector` değil).
2. "Aynı namespace'teyiz" güvenlik garantisi değildir — `default-deny-ingress` + role özel `allow` kuralları olmadan, aynı namespace'teki her pod birbirine serbestçe erişir.
3. `default-deny-ingress` Route trafiğini de keser: dışarıya açık katman (frontend) için router namespace'inden (`policy-group.network.openshift.io/ingress`) gelen trafiğe açıkça izin verilmelidir. Dışarı açılmaması gereken katmanlar (database) için böyle bir kural yazılmaz.
4. Testleri her zaman **ilgili rolün pod'u içinden** yapın — segmentasyona tabi olmayan (label'sız) bir test pod'u yanıltıcı "her şey kırık" sonuçları verir.
