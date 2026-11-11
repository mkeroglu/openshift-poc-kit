# 13 — East-West Trafik Kontrolü Demo Rehberi (Kısıtlama + Gözlemleme)

> 🇬🇧 [English](README.md) · [← 12 — Logging + LokiStack](../12-logging/README.tr.md) · [POC akışı](../../README.tr.md) · [14 — MultiCluster Observability →](../14-multicluster-observability/README.tr.md)

Bu doküman, cluster **içindeki servisler arası (east-west/yatay) trafiğin** hem **NetworkPolicy ile kısıtlanmasını** hem de **OVN ACL Logging** ile bu kararların (izin/red) **gerçek log kaydına düşerek gözlemlenmesini** anlatır. [08 — NetworkPolicy](../../C-network-and-security/08-network-policy/README.tr.md) ve [09 — Mikro-Segmentasyon](../../C-network-and-security/09-micro-segmentation/README.tr.md) rehberleri sadece **kısıtlamayı** kanıtlıyordu (bağlantı geçti/geçmedi); burada ayrıca **her bağlantı kararının (allow/drop) kimin tarafından, hangi policy yüzünden, hangi IP/port için verildiğinin** ham log satırı olarak görünür olduğu canlı olarak gösterilmiştir.

Tüm adımlar lab ortamında (OpenShift 4.22, **OVNKubernetes**) **`poc-eastwest-demo` namespace'inde canlı test edilmiştir** — gerçek OVN ACL audit log satırları, gerçek pod IP'leriyle birebir eşleştirilerek doğrulanmıştır. [12 — Logging](../12-logging/README.tr.md) kuruluysa aynı kayıtlar Loki'den merkezi olarak da sorgulanabilir (Bölüm 5.1).

Senaryo sırası:

1. Kavram — east-west trafik ve neden gözlemlemek gerekir
2. Demo ortamının kurulması (izinli kaynak, engellenecek kaynak, korunan hedef)
3. NetworkPolicy ile kısıtlama + ACL logging'i etkinleştirme
4. Trafik üretme ve doğrulama (bağlantı seviyesi)
5. **Gözlemleme**: gerçek OVN ACL audit log kayıtları
6. Gerçek tuzak: annotation'ın doğru yere yazılması
7. Temizlik

---

## 1. Kavram

- **North-south trafik**: cluster dışından içeri (Route/Ingress/LoadBalancer) veya içeriden dışarı giden trafik.
- **East-west trafik**: cluster **içindeki pod'lar arasındaki** (servisten servise, aynı veya farklı namespace) yatay trafik. Modern mikroservis mimarilerinde trafiğin büyük çoğunluğu buradadır — ve klasik "perimeter firewall" (sadece dış sınırı koru) yaklaşımı bunu görmez.
- **Kısıtlama** (NetworkPolicy — bkz. [08 — NetworkPolicy](../../C-network-and-security/08-network-policy/README.tr.md), [09 — Mikro-Segmentasyon](../../C-network-and-security/09-micro-segmentation/README.tr.md)): hangi pod'un hangi pod'a erişebileceğini tanımlar.
- **Gözlemleme** (bu doküman): kısıtlamanın **gerçekten çalıştığını**, **kimin denediğini** ve **ne zaman reddedildiğini** kanıtlayan audit iziydir. OVNKubernetes'te bu, **ACL Logging** özelliğiyle sağlanır — NetworkPolicy'nin ürettiği her OVN ACL kuralına (`allow`/`drop`) bir log seviyesi atanır, eşleşen her paket için node'da bir log satırı üretilir.

---

## 2. Demo Ortamının Kurulması

Üç pod: **izin verilen kaynak** (`app-a`), **engellenecek kaynak** (`app-c`), **korunan hedef** (`app-b`).

```yaml
# eastwest-demo.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: poc-eastwest-demo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-a-allowed
  namespace: poc-eastwest-demo
  labels: { role: allowed-source }
spec:
  replicas: 1
  selector: { matchLabels: { role: allowed-source } }
  template:
    metadata: { labels: { role: allowed-source } }
    spec:
      containers:
        - name: web
          image: busybox:1.36
          command: ["sh", "-c", "sleep 3600"]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-c-blocked
  namespace: poc-eastwest-demo
  labels: { role: blocked-source }
spec:
  replicas: 1
  selector: { matchLabels: { role: blocked-source } }
  template:
    metadata: { labels: { role: blocked-source } }
    spec:
      containers:
        - name: web
          image: busybox:1.36
          command: ["sh", "-c", "sleep 3600"]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-b-protected
  namespace: poc-eastwest-demo
  labels: { role: protected-target }
spec:
  replicas: 1
  selector: { matchLabels: { role: protected-target } }
  template:
    metadata: { labels: { role: protected-target } }
    spec:
      containers:
        - name: web
          image: busybox:1.36
          command: ["sh", "-c", "mkdir -p /tmp/www && echo APP-B-OK > /tmp/www/index.html && httpd -f -p 8080 -h /tmp/www"]
          ports: [{ containerPort: 8080 }]
---
apiVersion: v1
kind: Service
metadata: { name: app-b, namespace: poc-eastwest-demo }
spec: { selector: { role: protected-target }, ports: [{ port: 8080 }] }
```

```bash
oc apply -f eastwest-demo.yaml
```

---

## 3. NetworkPolicy ile Kısıtlama + ACL Logging

```yaml
# allow-only-app-a.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-only-app-a
  namespace: poc-eastwest-demo
spec:
  podSelector:
    matchLabels: { role: protected-target }
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector:
            matchLabels: { role: allowed-source }
      ports:
        - protocol: TCP
          port: 8080
```

```bash
oc apply -f allow-only-app-a.yaml
```

ACL logging'i **namespace** seviyesinde etkinleştirin (bkz. Bölüm 6 — kritik detay: annotation NetworkPolicy'ye değil, **Namespace**'e yazılır):

```bash
oc annotate namespace poc-eastwest-demo \
  'k8s.ovn.org/acl-logging={"deny":"alert","allow":"notice"}' --overwrite
```

- `deny` seviyesi: reddedilen bağlantılar için log severity'si (`alert`, `warning`, `notice`, `info`, `debug`).
- `allow` seviyesi: izin verilen bağlantılar için log severity'si.
- Namespace'teki **tüm** NetworkPolicy'ler ve EgressFirewall kuralları bu ayardan etkilenir.

---

## 4. Trafik Üretme ve Doğrulama (Bağlantı Seviyesi)

```bash
A_POD=$(oc get pod -n poc-eastwest-demo -l role=allowed-source -o jsonpath='{.items[0].metadata.name}')
C_POD=$(oc get pod -n poc-eastwest-demo -l role=blocked-source -o jsonpath='{.items[0].metadata.name}')

echo "app-a -> app-b (izinli olmali):"
oc exec "$A_POD" -n poc-eastwest-demo -- wget -qO- --timeout=3 app-b.poc-eastwest-demo.svc.cluster.local:8080

echo "app-c -> app-b (engellenmeli):"
oc exec "$C_POD" -n poc-eastwest-demo -- wget -qO- --timeout=3 app-b.poc-eastwest-demo.svc.cluster.local:8080
```

✅ **Gerçek çıktı:**

```
app-a -> app-b (izinli olmali):
APP-B-OK

app-c -> app-b (engellenmeli):
wget: download timed out
command terminated with exit code 1
```

Buraya kadar [08 — NetworkPolicy](../../C-network-and-security/08-network-policy/README.tr.md)/[09 — Mikro-Segmentasyon](../../C-network-and-security/09-micro-segmentation/README.tr.md) demolarıyla aynı — kısıtlama çalışıyor. Asıl fark bir sonraki bölümde.

---

## 5. Gözlemleme: Gerçek OVN ACL Audit Log Kayıtları

Kararların gerçekten **loglandığını** node üzerinde doğrulayın (`app-b`'nin çalıştığı node'da):

```bash
NODE=$(oc get pod -n poc-eastwest-demo -l role=protected-target -o jsonpath='{.items[0].spec.nodeName}')
oc debug node/$NODE --to-namespace=default -- chroot /host grep poc-eastwest-demo /var/log/ovn/acl-audit-log.log | tail -n 20
```

✅ **Gerçek çıktı** (pod IP'leriyle birebir eşleşiyor — `app-a=...96`, `app-c=...97`, `app-b=...98`; ikinci test turunda `.179/.180/.181` ile aynı sonuç):

```
...|INFO|name="NP:poc-eastwest-demo:allow-only-app-a:Ingress:0", verdict=allow, severity=notice, direction=to-lport:
    tcp,...,nw_src=10.131.3.96,nw_dst=10.131.3.98,...,tp_src=39036,tp_dst=8080,tcp_flags=syn
...|INFO|name="NP:poc-eastwest-demo:allow-only-app-a:Ingress:0", verdict=allow, severity=notice, direction=to-lport:
    tcp,...,nw_src=10.131.3.96,nw_dst=10.131.3.98,...,tcp_flags=ack
...|INFO|name="NP:poc-eastwest-demo:allow-only-app-a:Ingress:0", verdict=allow, severity=notice, direction=to-lport:
    tcp,...,nw_src=10.131.3.96,nw_dst=10.131.3.98,...,tcp_flags=psh|ack
...
...|INFO|name="NP:poc-eastwest-demo:Ingress", verdict=drop, severity=alert, direction=to-lport:
    tcp,...,nw_src=10.131.3.97,nw_dst=10.131.3.98,...,tp_src=60386,tp_dst=8080,tcp_flags=syn
...|INFO|name="NP:poc-eastwest-demo:Ingress", verdict=drop, severity=alert, direction=to-lport:
    tcp,...,nw_src=10.131.3.97,nw_dst=10.131.3.98,...,tcp_flags=syn
```

**Yorumu:**

- İlk grup (`verdict=allow`, `severity=notice`, ACL adı `NP:poc-eastwest-demo:allow-only-app-a:Ingress:0`): `app-a`'nın (`.96`) `app-b`'ye (`.98`) TCP handshake'inin **tamamı** (syn → ack → psh|ack → ack → fin|ack) — hangi NetworkPolicy'nin izin verdiği isimle birlikte görünüyor.
- İkinci grup (`verdict=drop`, `severity=alert`, ACL adı `NP:poc-eastwest-demo:Ingress` — genel/implicit default-deny): `app-c`'nin (`.97`) `app-b`'ye ulaşmaya çalışan **her SYN paketi tek tek reddedilip loglanıyor** (istemcinin tekrar tekrar denemesi TCP retransmisyonu olarak görünüyor).

**Bu, "gözlemlenir" gereksinimini tam olarak karşılıyor:** sadece "bağlantı koptu" demiyoruz, **kimin (kaynak IP), ne zamandan beri, hangi policy'ye takıldığını** ham log satırından okuyabiliyoruz.

### 5.1 Merkezi gözlem: ACL kayıtlarını Loki'den sorgulamak

OVN ACL audit logları OpenShift Logging'de **`audit`** log tipinin parçasıdır (`infrastructure` değil). [12 — Logging](../12-logging/README.tr.md) rehberindeki `ClusterLogForwarder` `audit` girdisini zaten topladığı için **ek bir ayar gerekmeden** Loki'ye akarlar:

```bash
TOKEN=$(oc create token log-reader -n openshift-logging --duration=1h)
ROUTE=$(oc get route logging-loki -n openshift-logging -o jsonpath='https://{.spec.host}')
curl -sk -H "Authorization: Bearer $TOKEN" \
  --data-urlencode 'query={log_type="audit"} |= "NP:poc-eastwest-demo"' \
  --data-urlencode "start=$(($(date +%s)-1800))000000000" --data-urlencode "end=$(date +%s)000000000" \
  "$ROUTE/api/logs/v1/audit/loki/api/v1/query_range"
```

✅ **Gerçek çıktı:** Bu namespace için **7 kayıt** (5 `verdict=allow`, 2 `verdict=drop`); node'daki dosyayla aynı. Stream etiketleri: `log_type=audit`, `k8s_node_name=<node>`. Bu kayıtlarda `log_source` etiketi **yoktur**; ACL kayıtlarını ayırmak için `|= "NP:<namespace>"` ya da `|= "verdict=drop"` gibi metin filtreleri kullanılır. Böylece node'lara tek tek `oc debug` ile bakmadan, tüm cluster'daki reddedilen east-west trafik tek sorguyla izlenebilir ve üzerine alarm kurulabilir.

---

## 6. Gerçek Tuzak: Annotation'ın Doğru Yere Yazılması

İlk denemede `k8s.ovn.org/acl-logging` annotation'ını **NetworkPolicy** objesinin üzerine yazdık — bu **sessizce hiçbir etki yapmadı**: kısıtlama çalıştı (`app-c` engellendi) ama node'daki log dosyasında bu namespace için **0** satır vardı (✅ ikinci test turunda da aynı sonuç; hata mesajı yok). Doğrusu: annotation **Namespace** objesine yazılmalı:

```bash
# YANLIŞ (etkisiz, hata da vermez):
oc annotate networkpolicy allow-only-app-a -n poc-eastwest-demo 'k8s.ovn.org/acl-logging=...'

# DOĞRU:
oc annotate namespace poc-eastwest-demo 'k8s.ovn.org/acl-logging=...'
```

Bu, sessizce hiçbir uyarı vermediği için tespit edilmesi zor bir tuzak — dosyanın gerçekten dolduğunu (`wc -l`) doğrulamadan "loglama açık" varsaymayın.

---

## 7. Temizlik

```bash
oc delete project poc-eastwest-demo
```

---

## Özet Tablo

| Bileşen | Rolü |
|---|---|
| `NetworkPolicy` | East-west trafiği **kısıtlar** (izin/red kararı) |
| `Namespace` annotation `k8s.ovn.org/acl-logging` | O namespace'teki tüm NetworkPolicy/EgressFirewall kararlarının **loglanmasını** açar |
| `/var/log/ovn/acl-audit-log.log` (node üzerinde) | Her allow/drop kararının ham kaydı — kaynak/hedef IP, port, TCP bayrağı, hangi policy |
| `ClusterLogForwarder` **audit** input | Bu logları merkezi Loki'ye taşır (ek ayar gerekmez) — bkz. [12 — Logging](../12-logging/README.tr.md) |

**Altın kurallar:**
1. ACL logging annotation'ı **Namespace**'e yazılır, NetworkPolicy'ye değil — yanlış yere yazmak sessizce hiçbir şey yapmaz.
2. `deny`/`allow` severity'leri **ayrı ayrı** ayarlanır — genelde `deny: alert` (dikkat çekmesi gereken, beklenmedik reddedilen trafik) + `allow: notice`/`info` (normal, hacimli trafik için daha düşük öncelik) mantıklı bir varsayılan.
3. Loglar node'un yerel dosya sistemindedir (`/var/log/ovn/acl-audit-log.log`); [12 — Logging](../12-logging/README.tr.md)'deki ClusterLogForwarder **`audit`** girdisi bunları otomatik olarak Loki'ye taşır.
4. Kısıtlama (NetworkPolicy) ile gözlemleme (ACL logging) **birbirinden bağımsız** açılıp kapatılan iki ayrı mekanizmadır — biri diğerini otomatik getirmez.
