# 05 — CI/CD (Pipelines + GitOps)

> 🇬🇧 [English](README.md) · [← 04 — Image Yönetimi](../../A-platform-foundation/04-image-management/README.tr.md) · [POC akışı](../../README.tr.md) · [06 — Blue/Green Deployment →](../06-blue-green-deployment/README.tr.md)

Bu bölümde OpenShift üzerinde CI/CD ve deployment stratejileri ele alınacaktır. Tüm adımlar lab ortamında (OpenShift 4.22, OpenShift GitOps 1.22, OpenShift Pipelines 1.23) **uçtan uca canlı test edilmiştir**; GitOps testi bu reponun kendisi (GitHub, `main`) kaynak alınarak yapılmıştır:

- [x] GitOps deployment
- [x] Pipeline otomasyonu
- [x] Blue/Green deployment
- [x] Canary deployment

---

## GitOps Deployment

### GitOps nedir?

GitOps, bir cluster'ın istenen durumunun (desired state) bir **Git deposunda** deklaratif YAML/Kustomize/Helm manifestoları olarak tutulduğu ve bir **operator/controller**'ın (bu POC'de: **Argo CD**) bu depoyu sürekli izleyip cluster'ın gerçek durumunu otomatik olarak depodakiyle eşitlediği bir dağıtım modelidir. Temel ilkeler:

- **Tek doğruluk kaynağı (single source of truth):** Cluster'da ne çalıştığı, Git geçmişinden okunabilir olmalıdır. `oc apply` ile elle yapılan değişiklikler yerine, değişiklik Git'e commit edilir.
- **Deklaratif:** Depo, "ne yapılacağını" değil "son halin ne olması gerektiğini" tanımlar.
- **Otomatik senkronizasyon + self-healing:** Cluster'daki bir kaynak elle değiştirilir/silinirse, GitOps controller'ı bunu tekrar Git'teki haliyle eşitler (drift'i otomatik düzeltir).
- **Denetlenebilirlik:** Her değişiklik bir Git commit'idir — kim, ne zaman, neyi değiştirdi net bir şekilde görülebilir.

Araç: **Argo CD** (OpenShift'te **Red Hat OpenShift GitOps** operatörü olarak dağıtılır).

### Operatör kurulumu

**a) YAML ile (referans — cluster genelinde etkili bir kaynaktır, uygulamadan önce zaten kurulu olup olmadığını kontrol edin):**

```bash
oc get csv -A | grep -i gitops   # zaten kurulu mu?
```

Kurulu değilse `gitops-operator-subscription.yaml`:

```bash
oc apply -f gitops-operator-subscription.yaml
oc get csv -n openshift-operators | grep gitops
oc get ns openshift-gitops openshift-gitops-operator
```

Subscription oluşturulduktan sonra operatör otomatik olarak `openshift-gitops-operator` namespace'ini ve varsayılan bir Argo CD instance'ını (`openshift-gitops` namespace'inde) kurar.

**b) Web Console (OperatorHub) ile:**

1. **Administrator** görünümünde **Operators > OperatorHub**'a gidin.
2. Arama kutusuna **"Red Hat OpenShift GitOps"** yazın, sonucu seçin.
3. **Install** butonuna tıklayın.
4. Varsayılan ayarlarla (**Update Channel: latest**, **Installation Mode: All namespaces**, **Automatic** approval) **Install**'a devam edin.
5. Kurulum tamamlandığında sol menüde bir **GitOps** simgesi (üst navigasyon çubuğunda, kutucuklar menüsünde) belirir; bu, Argo CD instance'ının Route'una kısayoldur.

### Argo CD arayüzüne erişim

```bash
oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='{.spec.host}{"\n"}'
```

Varsayılan kullanıcı adı `admin`; şifre otomatik oluşturulan bir secret'ta tutulur:

```bash
oc extract secret/openshift-gitops-cluster -n openshift-gitops --to=- --keys=admin.password
```

CLI ile giriş:

```bash
argocd login <argocd-route-host> --username admin --password <yukarıdaki-şifre> --insecure
```

### Argo CD'ye Repository ekleme (arayüzden)

1. Argo CD UI'da sol menüden **Settings > Repositories**'e gidin.
2. **+ CONNECT REPO** butonuna tıklayın.
3. Bağlantı yöntemini seçin:
   - **VIA HTTPS**: Repository URL (`https://.../repo.git`), gerekiyorsa Username + Password/Token.
   - **VIA SSH**: Repository URL (`git@host:org/repo.git`) ve bir SSH private key.
4. **CONNECT** ile kaydedin — bağlantı başarılıysa depo listede yeşil bir bağlantı durumuyla görünür.

> Private bir depo için: HTTPS'te genelde bir **personal access token** (parola yerine), SSH'da ise repo'ya salt-okunur erişimi olan bir **deploy key** kullanılması önerilir.

### Application oluşturma — YAML ile

`argocd-application.yaml` (bu repodaki `app/overlays/dev` dizinini hedefler):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gitops-demo-dev
  namespace: openshift-gitops
spec:
  project: default
  source:
    repoURL: https://github.com/mkeroglu/openshift-poc-kit.git
    targetRevision: main
    path: B-application-delivery/05-ci-cd/app/overlays/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: ocp-poc-gitops-dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

```bash
# bu repoyla doğrudan çalışır; kendi değişikliklerinizi denemek için repoURL'i fork'unuza çevirin:
oc apply -f argocd-application.yaml
oc get applications.argoproj.io -n openshift-gitops
```

✅ **Gerçek çıktı:** Application oluşturulduktan **5 sn** sonra `Sync: Synced, Health: Healthy`, revizyon = repodaki son commit. `CreateNamespace=true` ile `ocp-poc-gitops-dev` namespace'i oluştu; Deployment `1/1`, Service ve edge Route geldi (`https://...` → `200`). Aynı repodan `overlays/prod`'u hedefleyen ikinci bir Application `ocp-poc-gitops-prod`'a **3/3** replika ile açıldı.

> **Yetki hatası alırsanız:** Varsayılan `openshift-gitops` Argo CD instance'ı bazı kurulumlarda sadece kendi namespace'ini ve `argocd.argoproj.io/managed-by=openshift-gitops` etiketli namespace'leri yönetebilir. Sync `... is forbidden` hatasıyla düşerse hedef namespace'i etiketleyin: `oc label namespace <ns> argocd.argoproj.io/managed-by=openshift-gitops`. (`CreateNamespace=true` ile oluşan namespace'e etiket `managedNamespaceMetadata` ile verilebilir.)

> **Not:** `Application` kaynağı hem `argoproj.io` hem farklı bir operatörden gelen `app.k8s.io` API grubunda aynı isimle (`applications`) bulunabilir. Belirsizlik olursa tam kaynak adını kullanın: `oc get applications.argoproj.io`.

### Application oluşturma — arayüzden

1. Argo CD UI ana sayfasında **+ NEW APP** butonuna tıklayın.
2. **GENERAL**: Application Name, Project (`default`), Sync Policy (`Automatic` veya `Manual`).
3. **SOURCE**: az önce eklediğiniz Repository URL'i seçin, Revision (örn. `main`), Path (örn. `B-application-delivery/05-ci-cd/app/overlays/dev`).
4. **DESTINATION**: Cluster URL (`https://kubernetes.default.svc` — aynı cluster), Namespace (örn. `ocp-poc-gitops-dev`).
5. Kustomize path'i otomatik algılanır (ekstra bir alan gerekmez).
6. **CREATE** ile Application'ı oluşturun; ardından **SYNC** butonuna basarak ilk senkronizasyonu tetikleyin (Automatic policy seçtiyseniz bu otomatik olur).

### Repo içeriği nasıl olmalı

GitOps ile yönetilecek bir repo, tipik olarak **Kustomize base + overlay** düzeninde olmalıdır — bu repodaki `app/` dizini bunun canlı örneğidir:

```
app/
├── base/
│   ├── kustomization.yaml   # ortak kaynaklar
│   ├── deployment.yaml
│   ├── service.yaml
│   └── route.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml   # base'i referans alır, namespace + replika sayısını ortama göre değiştirir
    └── prod/
        └── kustomization.yaml
```

- **`base/`**: ortamdan bağımsız, ortak manifestolar.
- **`overlays/<env>/`**: sadece o ortama özgü farkları (namespace, replika sayısı, image tag, resource limitleri vb.) `kustomization.yaml` üzerinden patch'ler; `base/`'i değiştirmez.
- Her Argo CD `Application`'ı, `spec.source.path` alanıyla bu overlay dizinlerinden **birini** hedefler (örn. dev ortamı için `app/overlays/dev`, prod için `app/overlays/prod`) — böylece aynı uygulamanın farklı ortamları aynı repodan, farklı `Application` kaynaklarıyla yönetilir.

### Sync (senkronizasyon)

**Manuel vs Otomatik:**

- `syncPolicy` tanımlanmazsa Application **manuel** kalır — Git'teki değişiklik cluster'a **otomatik yansımaz**, UI'da **SYNC** butonuna basmak (veya `argocd app sync <ad>` / `oc patch` ile) gerekir.
- `syncPolicy.automated` tanımlıysa (yukarıdaki `argocd-application.yaml`'daki gibi), Argo CD Git'teki her değişikliği **otomatik olarak** cluster'a uygular.

**Önemli alt seçenekler:**

| Alan | Ne yapar |
|---|---|
| `automated.selfHeal: true` | Cluster'daki bir kaynak Git'teki tanımdan **elle** saptırılırsa (örn. biri `oc edit`/`oc delete` yaparsa), Argo CD bunu otomatik olarak Git'teki haline geri döndürür. |
| `automated.prune: true` | Git'ten bir kaynak **silinirse**, cluster'daki karşılığı da otomatik silinir. |
| `syncOptions: [CreateNamespace=true]` | Hedef namespace yoksa otomatik oluşturulur. |

**Canlı doğrulanan `selfHeal` davranışı** — ✅ **Gerçek çıktı:**

| Elle yapılan değişiklik | Argo CD'nin tepkisi |
|---|---|
| `oc delete svc gitops-demo` | Service **2 sn** içinde Git'teki haliyle yeniden oluşturuldu |
| `oc scale deploy/gitops-demo --replicas=4` (Git'te 1) | Replika **8 sn** içinde tekrar 1'e indirildi |

Her iki durumda da Application `Synced/Healthy` kaldı; elle müdahaleye gerek olmadı.

**Manuel sync tetikleme:**

```bash
argocd app sync gitops-demo-dev
# veya UI'dan: Application kartına tıklayıp SYNC butonuna basmak
```

**Durumu kontrol etme:**

```bash
oc get applications.argoproj.io gitops-demo-dev -n openshift-gitops -o jsonpath='Sync: {.status.sync.status}, Health: {.status.health.status}{"\n"}'
```

`Sync` alanı `Synced`/`OutOfSync`, `Health` alanı `Healthy`/`Progressing`/`Degraded` gibi değerler alır.

### Temizlik

```bash
oc delete applications.argoproj.io gitops-demo-dev -n openshift-gitops
oc delete namespace ocp-poc-gitops-dev ocp-poc-gitops-prod --ignore-not-found
```

## Pipeline Otomasyonu

### Tekton / OpenShift Pipelines nedir?

**OpenShift Pipelines**, Kubernetes-native bir CI/CD motoru olan **Tekton**'un OpenShift'e entegre dağıtımıdır. Kavramlar:

- **Task**: tek bir iş birimi (örn. "git'ten kod çek", "imaj build et"). Bir veya daha fazla **Step** (her biri ayrı bir container) içerir.
- **Pipeline**: birden fazla Task'ı sıralı (`runAfter`) veya paralel bağlayan iş akışı tanımı.
- **PipelineRun**: bir Pipeline'ın parametrelerle somutlaştırılmış **tek bir çalıştırılma örneği** (her PipelineRun kendi Pod'larını oluşturur).
- **TaskRun**: bir Task'ın somut çalıştırılma örneği (bir PipelineRun içinde otomatik oluşur).
- **Workspace**: Task'lar arasında paylaşılan disk alanı — örn. `git-clone` Task'ının indirdiği kaynak kodu, `buildah` Task'ının okuyabilmesi için ortak bir workspace'e yazılır.
- **Trigger** (`EventListener` + `TriggerBinding` + `TriggerTemplate`): bir Git webhook'u geldiğinde otomatik olarak yeni bir PipelineRun oluşturan mekanizma.

### Operatör kurulumu

**a) YAML ile (referans — cluster genelinde etkili, önce kurulu olup olmadığını kontrol edin):**

```bash
oc get csv -A | grep -i pipelines
```

Kurulu değilse:

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-pipelines-operator
  namespace: openshift-operators
spec:
  channel: latest
  name: openshift-pipelines-operator-rh
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  installPlanApproval: Automatic
```

```bash
oc apply -f <yukarıdaki-dosya>
oc get ns openshift-pipelines
```

**b) Web Console (OperatorHub) ile:**

1. **Administrator** görünümünde **Operators > OperatorHub**'a gidin.
2. **"Red Hat OpenShift Pipelines"** operatörünü arayıp seçin.
3. **Install** → varsayılan ayarlarla (**All namespaces**, **Automatic** approval) devam edin.
4. Kurulum tamamlandığında `openshift-pipelines` namespace'i ve içinde hazır **Task** kütüphanesi otomatik oluşur.

### Hazır Task'lar

Eskiden `ClusterTask` adında ayrı bir kaynak tipi vardı; güncel sürümlerde bu **deprecate edildi**. Yerine, sık kullanılan Task'lar (`git-clone`, `buildah`, `openshift-client`, `s2i-*`, `maven`, `kn`, vb.) doğrudan **`openshift-pipelines` namespace'inde** normal `Task` kaynağı olarak kurulu gelir ve bir Pipeline içinden **`resolver: cluster`** ile referans verilir:

```bash
oc get tasks -n openshift-pipelines
```

```yaml
taskRef:
  resolver: cluster
  params:
    - name: kind
      value: task
    - name: name
      value: git-clone
    - name: namespace
      value: openshift-pipelines
```

### Örnek Pipeline: build-and-deploy

`pipeline-build-deploy.yaml` — üç adımlı, gerçekçi bir "clone → build & push → deploy" akışı:

1. **fetch-source** (`git-clone` Task) — parametre olarak verilen Git deposunu `shared-workspace`'e klonlar.
2. **build-and-push** (`buildah` Task, `runAfter: fetch-source`) — klonlanan kaynaktaki `Dockerfile`'ı build edip `IMAGE` parametresinde belirtilen adrese (örn. internal registry) push eder.
3. **deploy** (`openshift-client` Task, `runAfter: build-and-push`) — hedef Deployment'ı `oc rollout restart` ile yeniden başlatarak yeni imajı devreye alır.

Pipeline parametreleri: `git-url`, `git-revision`, `context-dir` (Dockerfile'ın bulunduğu alt dizin, kök için `.`), `image`, `deployment-name`.

```bash
oc apply -f pipeline-build-deploy.yaml
```

### Hedef uygulamayı önceden oluşturma

Pipeline'ın `deploy` adımı `oc rollout restart` çalıştırdığı için, hedef **Deployment'ın pipeline ilk çalışmadan önce var olması** gerekir (rollout restart var olmayan bir Deployment'ı oluşturmaz, sadece mevcut olanı yeniden başlatır). `pipeline-target-app.yaml` bunu sağlar — namespace'inizi yazıp uygulayın:

```bash
oc apply -f pipeline-target-app.yaml
# ilk anda pod'un ImagePullBackOff olması normaldir — imaj henüz hiç push edilmedi
```

> Container'da `imagePullPolicy: Always` kullanıldı çünkü imaj her build'de aynı `:latest` tag'ini kullanıyor — bu olmadan node, önceden çektiği (eski) imajı önbellekten kullanmaya devam edebilir.

### Yeni namespace'te ilk çalıştırmadan önce: `pipelines-scc` kontrolü

**Yepyeni bir namespace'te** Pipeline'ı oluşturduktan hemen sonra PipelineRun'ı tetiklerseniz, `fetch-source` adımı `Permission denied` (`/workspace/output/.git`) veya `build-and-push` adımı `.docker`/`.config` yazma hatasıyla başarısız olabilir. Sebep: `git-clone` ve `buildah` Task'ları `pipelines-scc` SCC'siyle çalışmak üzere tasarlanmış, ama OpenShift Pipelines operatörü bu namespace için gereken `pipeline` ServiceAccount'unu ve `pipelines-scc-rolebinding`'i **Pipeline kaynağı oluşturulduktan birkaç saniye sonra** otomatik olarak provision ediyor — bu süre dolmadan PipelineRun tetiklenirse pod'lar daha kısıtlayıcı bir SCC'ye (`restricted-v2`) düşüyor ve bu hataları veriyor.

`oc apply -f pipeline-build-deploy.yaml` sonrasında, PipelineRun'ı tetiklemeden önce binding'in oluştuğunu doğrulayın:

```bash
oc get rolebinding pipelines-scc-rolebinding -n <namespace>
```

Komut `NotFound` dönerse birkaç saniye bekleyip tekrar deneyin (✅ canlı testte binding, Pipeline oluşturulduktan **8 sn** sonra geldi). Binding görünmüyorsa veya sorun devam ediyorsa, `pipeline` ServiceAccount'una `pipelines-scc`'yi elle bağlayarak da çözebilirsiniz:

```bash
oc adm policy add-scc-to-user pipelines-scc -z pipeline -n <namespace>
```

### PipelineRun ile çalıştırma (CLI)

`pipelinerun.yaml` — `image` alanındaki namespace'i kendi namespace'inizle değiştirip:

```bash
oc create -f pipelinerun.yaml
oc get pipelinerun -w
```

Log izleme:

```bash
# tkn CLI kuruluysa (kurulum: `tkn` binary'sini OpenShift Pipelines CLI olarak indirin):
tkn pipelinerun logs -f --last

# tkn yoksa doğrudan oc ile:
oc get pods -l tekton.dev/pipelineRun=<pipelinerun-adi>
oc logs -f <pod-adi> --all-containers
```

✅ **Gerçek çıktı:** `Succeeded: Tasks Completed: 3 (Failed: 0, Cancelled 0)`, toplam **164 sn**:

| Task | Süre |
|---|---|
| `fetch-source` (git-clone) | 39 sn |
| `build-and-push` (buildah → internal registry) | 1 dk 50 sn |
| `deploy` (`oc rollout restart` + status) | 8 sn |

`pipelines-demo` ImageStream'i oluştu; başlangıçta `ImagePullBackOff`'ta bekleyen Deployment yeni imajla `Running` oldu. Doğrulama: `curl http://<route>/vote` → `{"a":0,"b":0}` (`200`). (Bu örnek uygulamanın kök yolu `/` `404` döner; bu beklenen bir davranış.)

### Arayüzden (Web Console) çalıştırma

1. **Developer** görünümünde sol menüden **Pipelines**'e gidin.
2. `build-and-deploy` Pipeline'ına tıklayın, sağ üstten **Start**'ı seçin.
3. Açılan formda parametreleri (`git-url`, `git-revision`, `context-dir`, `image`, `deployment-name`) ve Workspace için bir **VolumeClaimTemplate** (boyut, örn. 1Gi) girin.
4. **Start** ile PipelineRun'ı tetikleyin — ilerleme, her Task için ayrı bir sütun/adım olarak görsel şekilde (DAG görünümü) izlenebilir; bir adıma tıklayarak canlı log akışını görebilirsiniz.

### Triggers ile otomatik tetikleme (webhook)

Her commit'te pipeline'ı elle başlatmak yerine, Git sağlayıcısının (GitHub/GitLab/Bitbucket/Gitea) **webhook**'u ile otomatik tetikleme yapılır:

- **TriggerBinding**: webhook payload'ındaki alanları (`body.repository.clone_url`, `body.after`) parametrelere eşler.
- **TriggerTemplate**: bu parametrelerle oluşturulacak PipelineRun'ın şablonu.
- **EventListener**: bir Service/Route üzerinden webhook'u dinler. **Interceptor** ile olay tipi filtrelenir (örn. sadece `push`) ve webhook imzası doğrulanabilir.

Akış: `Git push → webhook → EventListener Route → interceptor (filtre/imza) → TriggerBinding → TriggerTemplate → yeni PipelineRun`.

`triggers.yaml` bunların hepsini içerir (TriggerTemplate'teki `REPLACE_ME_NAMESPACE`'i kendi namespace'inizle değiştirin):

```bash
NS=<namespace>
# EventListener'ın cluster kapsamındaki interceptor'lara erişimi için (EventListener'dan ÖNCE oluşturun):
oc create clusterrolebinding $NS-trigger-sa-clusterroles \
  --clusterrole=tekton-triggers-eventlistener-clusterroles --serviceaccount=$NS:pipeline-trigger-sa
sed "s/REPLACE_ME_NAMESPACE/$NS/" triggers.yaml | oc apply -n $NS -f -
oc get eventlistener build-and-deploy-listener -n $NS        # READY=True
oc expose svc el-build-and-deploy-listener -n $NS            # webhook adresi
oc get route el-build-and-deploy-listener -n $NS -o jsonpath='{.spec.host}'
```

Git sağlayıcısında webhook: **Payload URL** = `http://<yukarıdaki route>`, **Content type** = `application/json`, **Events** = `push`. Webhook imzası doğrulanacaksa bir secret oluşturup interceptor'a ekleyin:

```yaml
interceptors:
  - ref: {name: github}
    params:
      - name: secretRef
        value: {secretName: github-webhook-secret, secretKey: token}
      - name: eventTypes
        value: ["push"]
```

> ClusterRoleBinding EventListener'dan sonra oluşturulursa EventListener pod'u ilk başlatmada yetki hatası alıp yeniden başlar ve `READY` birkaç saniye `False (MinimumReplicasUnavailable)` görünür. Sırayla oluşturmak bunu önler.

**Test (Git sağlayıcısı olmadan, webhook'u taklit ederek):**

```bash
EL=$(oc get route el-build-and-deploy-listener -n $NS -o jsonpath='{.spec.host}')
# push dışı olay -> filtrelenmeli
curl -s -X POST http://$EL -H 'Content-Type: application/json' -H 'X-GitHub-Event: issues' -d '{}'
# push olayı -> PipelineRun oluşmalı
curl -s -X POST http://$EL -H 'Content-Type: application/json' -H 'X-GitHub-Event: push' \
  -d '{"after":"master","repository":{"clone_url":"https://github.com/openshift/pipelines-vote-api.git"}}'
oc get pipelinerun -n $NS
```

✅ **Gerçek çıktı:** İki istek de `{"eventListener":"build-and-deploy-listener",...,"eventID":"..."}` döndü (EventListener isteği kabul eder, filtreleme arkada yapılır). Sadece `push` olayı için yeni bir PipelineRun (`build-and-deploy-webhook-9nxxz`) oluştu; parametreleri payload'dan geldi (`https://github.com/openshift/pipelines-vote-api.git @ master`) ve **55 sn**'de `Succeeded` oldu. `issues` olayı PipelineRun oluşturmadı.

### Temizlik

```bash
oc delete pipelinerun -l tekton.dev/pipeline=build-and-deploy
oc delete -f triggers.yaml
oc delete clusterrolebinding <namespace>-trigger-sa-clusterroles
oc delete -f pipeline-target-app.yaml
oc delete -f pipeline-build-deploy.yaml
```

---

## Blue/Green Deployment

Ayrı bir dokümana taşındı — bkz. **[06 — Blue/Green Deployment](../06-blue-green-deployment/README.tr.md)** (kavram, YAML'lar, canlı test çıktıları ve router gecikmesi tuzağı orada).

---

## Canary Deployment

Ayrı bir dokümana taşındı — bkz. **[07 — Canary Deployment](../07-canary-deployment/README.tr.md)** (kavram, Route `alternateBackends`/`weight` yapılandırması, `%90/10 → %50/50 → %100/0` kademeli geçişin gerçek ölçülmüş istek dağılımlarıyla canlı testi).
