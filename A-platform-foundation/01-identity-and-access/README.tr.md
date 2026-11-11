# 01 — Kimlik Doğrulama ve Yetkilendirme (OAuth + RBAC)

> 🇬🇧 [English](README.md) · [POC akışı](../../README.tr.md) · [02 — Container Yönetimi →](../02-container-management/README.tr.md)

Bu rehber, kullanıcıların OpenShift'e **kurumsal kimlikleriyle** (LDAP/Active Directory) nasıl giriş yaptığını ve **rol bazlı yetkilendirme (RBAC)** ile kimin, hangi namespace'te, neyi yapabileceğinin nasıl sınırlandığını anlatır.

Tüm adımlar lab ortamında (OpenShift 4.22) **uçtan uca canlı test edilmiştir**. Aşağıdaki "✅ Gerçek çıktı" satırları bu testlerden alınmıştır. Ortama özgü değerler (LDAP sunucusu, base DN, grup adları) müşteri ortamına göre değiştirilmelidir.

Senaryo sırası:

1. Mevcut kimlik sağlayıcılarını görme
2. LDAP / Active Directory kimlik sağlayıcısı
3. LDAP grup senkronizasyonu
4. HTPasswd (yerel) kimlik sağlayıcısı
5. RBAC senaryosu: dev / prod, geliştirici / izleyici / namespace admin
6. Doğrulama: yetki matrisi ve gerçek işlemler
7. Ek sıkılaştırmalar (self-provisioner, kubeadmin)
8. Canlı testte görülenler
9. Temizlik

---

## 1. Mevcut Kimlik Sağlayıcıları

```bash
oc get oauth cluster -o jsonpath='{range .spec.identityProviders[*]}{.name} ({.type}){"\n"}{end}'
oc get co authentication            # Available=True, Progressing=False, Degraded=False olmalı
oc get users; oc get identities     # Girişi yapılmış kullanıcılar ve hangi IdP'den geldikleri
oc get groups
```

- Bir kullanıcı ilk kez giriş yaptığında OpenShift bir **`User`** ve ona bağlı bir **`Identity`** (`<idp-adı>:<kullanıcı-id>`) nesnesi oluşturur.
- Aynı cluster'da birden fazla IdP tanımlanabilir. Giriş ekranında her biri ayrı bir seçenek olarak görünür.

**Console:** **Administration → Cluster Settings → Configuration → OAuth** → **Identity providers**.

---

## 2. LDAP / Active Directory Kimlik Sağlayıcısı

**Ön koşullar:** LDAP sunucusunun adresi, kullanıcı aramak için bir bind kullanıcısı (salt okuma yeterli) ve LDAPS kullanılıyorsa sunucu sertifikasının CA'sı.

```bash
# Bind kullanıcısının şifresi ve LDAP sunucusunun CA sertifikası
oc create secret generic ldap-bind-password -n openshift-config --from-literal=bindPassword='<BIND_PASSWORD>'
oc create configmap ldap-ca -n openshift-config --from-file=ca.crt=./ldap-ca.crt
```

`ldap-idp-patch.json` (mevcut IdP'leri **silmeden** ekler):

```json
[
  {
    "op": "add",
    "path": "/spec/identityProviders/-",
    "value": {
      "name": "Corporate-LDAP",
      "type": "LDAP",
      "mappingMethod": "claim",
      "ldap": {
        "url": "ldaps://<LDAP_SERVER>:636/<USER_BASE_DN>?sAMAccountName?sub?(objectClass=person)",
        "bindDN": "<BIND_USER_DN>",
        "bindPassword": { "name": "ldap-bind-password" },
        "ca": { "name": "ldap-ca" },
        "insecure": false,
        "attributes": {
          "id": ["sAMAccountName"],
          "preferredUsername": ["sAMAccountName"],
          "name": ["cn"],
          "email": ["mail"]
        }
      }
    }
  }
]
```

```bash
oc patch oauth cluster --type=json --patch-file ldap-idp-patch.json
oc get co authentication -w        # Progressing=True -> False (oauth pod'ları ~1 dk'da yenilenir)
```

> - **Active Directory için** kullanıcı adı alanı genellikle `sAMAccountName`'dir. OpenLDAP'ta `uid` kullanılır.
> - URL'deki filtre (`(objectClass=person)`) giriş yapabilecek kullanıcıları sınırlar. Örneğin sadece belirli bir grubun üyeleri için: `(&(objectClass=person)(memberOf=<GROUP_DN>))`.
> - İlk ayarda yanlış bir değer varsa giriş `401 Unauthorized` döner. Ayrıntı için: `oc logs -n openshift-authentication deploy/oauth-openshift`.

✅ **Gerçek çıktı:** Lab ortamında LDAP (Active Directory) IdP'si `claim` eşlemesiyle tanımlı. Daha önce giriş yapmış bir kullanıcının `User` nesnesinde `identities=["<LDAP-IdP-adı>:<base64-kullanıcı-id>"]` görüldü; LDAP'tan gelen kimliğin kullanıcıya bağlandığı doğrulandı.

**Console:** **OAuth → Add → LDAP** formu, ya da **YAML** sekmesi.

---

## 3. LDAP Grup Senkronizasyonu

LDAP IdP'si **sadece kimlik doğrulama** yapar; LDAP gruplarını OpenShift'e getirmez. Yetkileri LDAP gruplarına göre vermek için gruplar **senkronize edilmelidir**. İki yol vardır:

**a) `oc adm groups sync` (elle / CronJob ile)**

`ldap-sync.yaml`:

```yaml
kind: LDAPSyncConfig
apiVersion: v1
url: ldaps://<LDAP_SERVER>:636
bindDN: <BIND_USER_DN>
bindPassword: <BIND_PASSWORD>          # ya da: bindPassword: {file: /etc/secrets/bindPassword}
ca: ./ldap-ca.crt
augmentedActiveDirectory:
  groupsQuery:
    derefAliases: never
    pageSize: 0
  groupUIDAttribute: dn
  groupNameAttributes: [cn]
  usersQuery:
    baseDN: "<USER_BASE_DN>"
    scope: sub
    derefAliases: never
    filter: (objectClass=person)
    pageSize: 0
  userNameAttributes: [sAMAccountName]
  groupMembershipAttributes: ["memberOf:1.2.840.113556.1.4.1941:"]   # AD iç içe grupları
```

```bash
oc adm groups sync --sync-config=ldap-sync.yaml --whitelist=groups.txt            # dry-run (sadece gösterir)
oc adm groups sync --sync-config=ldap-sync.yaml --whitelist=groups.txt --confirm  # uygular
```

`groups.txt` içine sadece OpenShift'te kullanılacak grupların DN'leri yazılır. Bütün AD'yi senkronize etmek gereksiz binlerce grup oluşturur.

**b) Group Sync Operator (sürekli, zamanlanmış)**

OperatorHub'daki **Group Sync Operator** (`GroupSync` CR'ı) aynı işi belirli aralıklarla kendisi yapar. Lab ortamında grup senkronizasyonu bu operatörle çalışıyor.

✅ **Gerçek çıktı:** Lab ortamında 251 LDAP grubu OpenShift `Group` nesnesi olarak senkronize. Bir AD kullanıcısının 8 gruba üye olduğu ve bunlardan birine bağlı `ClusterRoleBinding` üzerinden yetki aldığı görüldü.

---

## 4. HTPasswd (Yerel) Kimlik Sağlayıcısı

LDAP'ın olmadığı ortamlar, acil durum hesapları ya da test kullanıcıları için kullanılır.

```bash
htpasswd -c -B -b users.htpasswd poc-admin  '<PASSWORD>'
htpasswd    -B -b users.htpasswd poc-dev    '<PASSWORD>'
htpasswd    -B -b users.htpasswd poc-viewer '<PASSWORD>'
oc create secret generic poc-htpasswd -n openshift-config --from-file=htpasswd=users.htpasswd

oc patch oauth cluster --type=json --patch-file htpasswd-idp-patch.json   # mevcut IdP'lere EKLER
oc get co authentication -w
```

Kullanıcı eklemek/çıkarmak için `users.htpasswd` güncellenip secret yeniden uygulanır: `oc set data secret/poc-htpasswd -n openshift-config --from-file=htpasswd=users.htpasswd`.

> ⚠️ **OAuth bir ACM Governance Policy ile yönetiliyorsa** eklenen IdP kısa süre içinde **geri silinir**.
>
> ✅ **Gerçek çıktı:** Lab ortamında patch `oauth.config.openshift.io/cluster patched` döndü, ama ~1 dk sonra IdP listesinde sadece LDAP IdP'si kaldı. Sebebi: `enforce` modunda ve `mustonlyhave` uyumluluk tipinde bir ACM Policy, OAuth'u sürekli olarak sadece LDAP IdP'sine eşitliyor. Kalıcı değişiklik **Policy'nin kendisinde** yapılmalıdır (ya da Policy geçici olarak `inform` moduna alınır). Bu yüzden aşağıdaki RBAC testleri gerçek giriş yerine **impersonation** ile yapıldı. Impersonation, API server'ın aynı yetkilendirme (RBAC) kararını verir.
>
> Kontrol için: `oc get policies.policy.open-cluster-management.io -A | grep -i oauth`

---

## 5. RBAC Senaryosu

`rbac.yaml` iki namespace ve üç rol profili oluşturur:

| Kullanıcı / Grup | `poc-rbac-dev` | `poc-rbac-prod` |
|---|---|---|
| `poc-developers` (poc-dev) | `edit` (her şeyi oluşturur/değiştirir) | `view` + özel `deployment-restarter` rolü |
| `poc-viewers` (poc-viewer) | `view` | `view` |
| `poc-admin` | `admin` | `admin` |

- **`view` / `edit` / `admin`**, OpenShift'in hazır ClusterRole'leridir. Bir `RoleBinding` ile **sadece o namespace'e** uygulanırlar.
- **`deployment-restarter`** namespace'e özel bir `Role`'dür: `deployments` üzerinde sadece `get/list/watch/patch`. Böylece geliştirici prod'da uygulamayı yeniden başlatabilir (`oc rollout restart` bir patch'tir), ama silemez ya da yeni kaynak oluşturamaz.
- `view` rolü **secret'ları göstermez**. `edit` gösterir; `admin` ayrıca namespace içinde yetki dağıtabilir.
- Gruplar burada elle (`Group` nesnesi) oluşturuldu. LDAP senkronizasyonu varsa RoleBinding'ler doğrudan senkronize edilen LDAP gruplarına yapılır (`name: <LDAP grup adı>`).

```bash
oc apply -f rbac.yaml
# Test için prod'da bir uygulama ve secret
oc create deployment web --image=registry.access.redhat.com/ubi9/httpd-24 -n poc-rbac-prod
oc create secret generic db-pass --from-literal=password=x -n poc-rbac-prod
```

**Console:** **User Management → Groups** (grup ve üyeler), **User Management → RoleBindings → Create binding** (namespace, rol, kullanıcı/grup). Proje sayfasında **Project access** sekmesinden de yetki verilebilir.

---

## 6. Doğrulama

**Gerçek kullanıcıyla (önerilen):** Her kullanıcı kendi şifresiyle giriş yapar (ayrı terminal ya da gizli tarayıcı penceresi):

```bash
oc login <API_URL> -u poc-dev -p '<PASSWORD>'
oc auth can-i create deployments -n poc-rbac-prod
```

**Impersonation ile (admin, tek terminalden):**

```bash
DEV="--as=poc-dev --as-group=poc-developers --as-group=system:authenticated"
oc auth can-i create deployments -n poc-rbac-dev $DEV
```

> ⚠️ Impersonation'da kullanıcının OpenShift `Group` üyelikleri **otomatik çözülmez**; grup `--as-group` ile açıkça verilmelidir.
>
> ✅ **Gerçek çıktı:** `oc auth can-i create deployments -n poc-rbac-dev --as=poc-dev` → **`no`**. Aynı komut `--as-group=poc-developers` ile → **`yes`**. Gerçek girişte OpenShift grup üyeliğini kendisi ekler.

**Yetki matrisi (`oc auth can-i ...`)** — ✅ **Gerçek çıktı:**

| İşlem | poc-dev | poc-viewer | poc-admin |
|---|---|---|---|
| dev: deployment oluştur | yes | no | yes |
| prod: deployment oluştur | **no** | no | yes |
| prod: deployment restart (patch) | **yes** | no | yes |
| prod: deployment sil | **no** | no | yes |
| prod: pod listele | yes | yes | yes |
| prod: secret oku | no | **no** | yes |
| prod: rolebinding oluştur | no | no | **yes** |
| başka namespace'te pod listele | no | no | **no** |
| node listele (cluster kapsamı) | no | no | **no** |

**Gerçek işlemler** — ✅ **Gerçek çıktı:**

```
[dev]    dev'de deployment oluştur : deployment.apps/app1 created
[dev]    prod'da deployment oluştur: deployments.apps is forbidden: User "poc-dev" cannot create resource "deployments"
[dev]    prod'da restart           : deployment.apps/web restarted
[dev]    prod'da silme             : deployments.apps "web" is forbidden: User "poc-dev" cannot delete resource "deployments"
[viewer] prod secret okuma         : secrets "db-pass" is forbidden: User "poc-viewer" cannot get resource "secrets"
[viewer] prod pod listesi          : web-679bff89b5-klwf4 ContainerCreating  web-f97f9bf9d-k7n88 Running
[viewer] prod'da pod silme         : pods "web-679bff89b5-klwf4" is forbidden: User "poc-viewer" cannot delete resource "pods"
[admin]  prod'da yetki verme       : rolebinding.rbac.authorization.k8s.io/viewer-extra created
[admin]  kendine cluster-admin     : clusterrolebindings.rbac.authorization.k8s.io is forbidden: User "poc-admin" cannot list resource "clusterrolebindings"
[admin]  başka ns'te pod listeleme : pods is forbidden: User "poc-admin" cannot list resource "pods"
```

Namespace admin'i kendi namespace'inde yetki dağıtabiliyor, ama kendini cluster-admin yapamıyor ve başka namespace'leri göremiyor.

**Kim, neyi yapabilir?**

```bash
oc adm policy who-can patch deployments -n poc-rbac-prod
```

✅ **Gerçek çıktı:** Kullanıcılar arasında `poc-admin`, gruplar arasında `poc-developers` (ve cluster-admin yetkili kurum grupları) listelendi.

---

## 7. Ek Sıkılaştırmalar (cluster geneli — POC'de uygulanmadı)

**Kullanıcıların kendi proje oluşturmasını kapatma** (varsayılanda açıktır):

```bash
oc get clusterrolebinding self-provisioners -o jsonpath='{.subjects}'   # system:authenticated:oauth
oc patch clusterrolebinding.rbac self-provisioners -p '{"subjects": null}'
oc patch clusterrolebinding.rbac self-provisioners -p '{"metadata":{"annotations":{"rbac.authorization.kubernetes.io/autoupdate":"false"}}}'
```

Bundan sonra proje oluşturma yetkisi sadece belirli bir gruba verilir: `oc adm policy add-cluster-role-to-group self-provisioner <grup>`.

**`kubeadmin` kullanıcısını kaldırma:** Kurumsal IdP üzerinden en az bir kullanıcıya `cluster-admin` verildikten ve bu kullanıcıyla giriş **doğrulandıktan** sonra:

```bash
oc delete secret kubeadmin -n kube-system
```

> Bu işlem geri alınamaz. Acil durumlar için `kubeconfig` (system:admin) dosyası güvenli bir yerde saklanmalıdır.

---

## 8. Canlı Testte Görülenler

- **ACM Policy OAuth'u geri yazıyor:** bkz. bölüm 4. OAuth değişikliği yapmadan önce `oc get policies.policy.open-cluster-management.io -A` ile OAuth'u yöneten bir Policy olup olmadığına bakın.
- **Impersonation grup üyeliğini çözmüyor:** bkz. bölüm 6.
- **Yanlış şifre / değişmiş şifre:** LDAP girişinde `Login failed (401 Unauthorized)`. AD hesabının kilitlenmemesi için tekrarlı denemeden önce şifreyi doğrulayın.

---

## 9. Temizlik

```bash
oc delete -f rbac.yaml
# HTPasswd IdP eklendiyse (index'i önce kontrol edin):
oc get oauth cluster -o jsonpath='{range .spec.identityProviders[*]}{.name}{"\n"}{end}' | cat -n
oc patch oauth cluster --type=json -p '[{"op":"remove","path":"/spec/identityProviders/<INDEX>"}]'
oc delete secret poc-htpasswd -n openshift-config
oc delete user poc-admin poc-dev poc-viewer --ignore-not-found
oc get identity -o name | grep poc-local | xargs -r oc delete
```

---

## Özet Tablo

| Bileşen | Rolü |
|---|---|
| `OAuth` (`cluster`) | Kimlik sağlayıcılarının (LDAP, HTPasswd, OIDC...) tanımlandığı tek nesne |
| `User` / `Identity` | İlk girişte oluşur; kullanıcıyı IdP'deki kimliğine bağlar |
| `Group` | Yetkilerin verildiği birim; LDAP'tan senkronize edilir ya da elle oluşturulur |
| `ClusterRole` (`view`/`edit`/`admin`) | Hazır yetki setleri |
| `Role` | Namespace'e özel, ince ayarlı yetki seti (örn. `deployment-restarter`) |
| `RoleBinding` | Bir rolü bir kullanıcıya/gruba **tek namespace'te** verir |
| `ClusterRoleBinding` | Bir rolü **tüm cluster'da** verir (dikkatli kullanılmalı) |
