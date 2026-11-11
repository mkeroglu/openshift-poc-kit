# OpenShift POC Kit

> 🇬🇧 [English](README.md)

**Gerçek bir cluster üzerinde uçtan uca çalıştırılmış 19 uygulamalı OpenShift POC rehberi.** Her rehberde YAML'lar, `oc` / `virtctl` komutları, Web Console adımları ve dönen **gerçek çıktılar** var. Bozulan kısımlar ve nasıl düzeltildikleri de dahil.

Çoğu OpenShift anlatımı "bunu uygula, çalışır" der ve orada biter. Bu rehberler devam eder: Blue/Green geçişinde router gecikmesi gerçekte ne kadar, egress NetworkPolicy neden DNS'i sessizce bozuyor, `oc get backup` neden yanlış kaynağı getiriyor, registry'den gelen Windows imajı neden boş ekranda açılıyor. Bu bulgular her rehberde **"✅ Gerçek çıktı"** olarak işaretlidir.

Test ortamı: OpenShift 4.22 · OpenShift Virtualization 4.22 · ODF 4.22 · OpenShift GitOps 1.22 · OpenShift Pipelines 1.23 · Logging 6.6 · OADP 1.6 · RHACS 4.11 · ACM · MTV 2.12 · bare-metal worker'lar, OVNKubernetes.

## Akış

Klasörler **uygulama sırasına göre** gruplanmış (A–E) ve numaralandırılmıştır (01–19). Sıra, rehberler arasındaki bağımlılıklara göre belirlendi: platform temeli → uygulama teslimi → ağ ve güvenlik → operasyon → sanallaştırma. Bir rehberin kurduğu kaynak (namespace, DPA, Loki vb.) sonraki rehberlerde kullanılıyor; bu yüzden sırayla gidilmesi önerilir.

| # | Rehber | Konu | Ön koşul / bağımlılık |
|---|---|---|---|
| **A** | **[Platform Temeli](A-platform-foundation)** | | |
| 01 | [Kimlik ve Yetki](A-platform-foundation/01-identity-and-access/README.tr.md) | LDAP/AD ve HTPasswd kimlik sağlayıcıları, grup senkronizasyonu, RBAC (dev/prod, rol bazlı yetki) | cluster-admin |
| 02 | [Container Yönetimi](A-platform-foundation/02-container-management/README.tr.md) | Namespace/Project, rollout/rollback, HPA, ResourceQuota, LimitRange | — (`ocp-poc` namespace'ini **bu rehber oluşturur**; 03, 04, 10, 12 bunu kullanır) |
| 03 | [Uygulama Senaryoları](A-platform-foundation/03-application-scenarios/README.tr.md) | Stateless, stateful (ODF), API/microservice | 02 (02'nin kotası kaldırılmış olmalı) |
| 04 | [Image Yönetimi](A-platform-foundation/04-image-management/README.tr.md) | Internal/external registry, private pull secret, ACS ile zafiyet taraması | 02, RHACS |
| **B** | **[Uygulama Teslimi](B-application-delivery)** | | |
| 05 | [CI/CD](B-application-delivery/05-ci-cd/README.tr.md) | OpenShift Pipelines + GitOps (Argo CD) | OpenShift Pipelines, OpenShift GitOps, bu repoya erişim |
| 06 | [Blue/Green Deployment](B-application-delivery/06-blue-green-deployment/README.tr.md) | İki Deployment + tek Route ile anlık geçiş | 05 (kavramsal devamı) |
| 07 | [Canary Deployment](B-application-delivery/07-canary-deployment/README.tr.md) | Ağırlıklı Route ile kademeli geçiş | 06 |
| **C** | **[Ağ ve Güvenlik](C-network-and-security)** | | |
| 08 | [NetworkPolicy](C-network-and-security/08-network-policy/README.tr.md) | Default deny, namespace/pod bazlı izin, egress | — |
| 09 | [Mikro-Segmentasyon](C-network-and-security/09-micro-segmentation/README.tr.md) | Katman/rol bazlı izolasyon | 08 |
| 10 | [Multus CNI](C-network-and-security/10-multus-cni/README.tr.md) | Pod'a ikinci ağ arayüzü (macvlan) | 02 |
| 11 | [Güvenlik Testleri](C-network-and-security/11-security-tests/README.tr.md) | SCC/PSA, image signing | ImagePolicy tüm node'lara MCP rollout'u yapar (~1,5 dk, reboot yok); cosign v2 ve `MatchRepository` gerekir |
| **D** | **[Operasyon](D-operations)** | | |
| 12 | [Logging + LokiStack](D-operations/12-logging/README.tr.md) | Application/audit logları → Loki (ODF S3) | ODF (Ceph RGW), 02 |
| 13 | [East-West Trafik Kontrolü](D-operations/13-east-west-traffic-control/README.tr.md) | NetworkPolicy + OVN ACL logging | 08, 09, 12 |
| 14 | [MultiCluster Observability](D-operations/14-multicluster-observability/README.tr.md) | ACM Observability (Thanos, ODF S3) | ACM (MultiClusterHub), ODF |
| 15 | [ODF Performans Testi](D-operations/15-odf-performance-test/README.tr.md) | fio ile IOPS / throughput / latency | ODF |
| 16 | [OADP Backup/Restore](D-operations/16-oadp/README.tr.md) | Velero ile namespace yedekleme/geri yükleme (ODF S3) | ODF (Ceph RGW). `dpa-odf`'i **bu rehber kurar**; 17 kullanır |
| **E** | **[Virtualization](E-virtualization)** | | |
| 17 | [OpenShift Virtualization](E-virtualization/17-virtualization/README.tr.md) | Template, ISO, golden image, live migration, snapshot, Multus VLAN, affinity, VM yedekleme | OpenShift Virtualization, ODF, NMState, 16 |
| 17.1 | ↳ [Windows Boot Source](E-virtualization/17-virtualization/windows/README.tr.md) | Windows template'leri için imaj sağlamanın 4 yolu | 17, (yöntem 3 için) OpenShift Pipelines |
| 18 | [MTV ile Taşıma](E-virtualization/18-mtv/README.tr.md) | VMware vSphere / OVA'dan VM taşıma (provider, haritalar, plan) | Migration Toolkit for Virtualization, 17 |
| 19 | [Node Kaybında Otomatik Kurtarma](E-virtualization/19-node-health-check/README.tr.md) | NodeHealthCheck + Self Node Remediation: çöken node'u fence edip VM'leri başka node'da açma | 17; worker node'ları (control plane hariç) |

## POC Öncesi Kontrol

Gün başında cluster'ın ve kullanılacak operatörlerin hazır olduğunu doğrulayın:

```bash
oc get clusterversion
oc get nodes
oc get co | awk 'NR==1 || $3!="True" || $4!="False" || $5!="False"'   # sorunlu ClusterOperator'lar
oc get storagecluster -n openshift-storage                               # ODF: Ready
oc get csv -A --no-headers -o custom-columns=CSV:.metadata.name,PHASE:.status.phase \
  | grep -iE '^(openshift-pipelines|openshift-gitops|rhacs|loki|cluster-logging|advanced-cluster|oadp|kubevirt|kubernetes-nmstate)' | sort -u
```

## Planlama Notları

- **Uzun sürenleri önceden başlatın:** LokiStack (12), MultiCluster Observability (14) kurulumları ve Windows imaj pipeline'ı (17.1, ~31 dk) arka planda kendiliğinden ilerler. Gün başında başlatılırsa sırası geldiğinde hazır olurlar. Windows ISO'su (~5 GB) önceden indirilmelidir.
- **Image signing (11):** `ImagePolicy` tüm node'larda MachineConfig rollout'u başlatır. Test cluster'ında yalnızca CRI-O reload yapıldı, reboot olmadı (~1,5 dk). Yine de değişiklik tüm node'lara dağıtıldığı için canlı ortamda bakım penceresi önerilir.
- **Paylaşımlı kaynaklar:** 16 ve 17, cluster genelindeki bazı kaynakları (DPA, `openshift-cnv`, `openshift-virtualization-os-images`) etkileyebilir. Rehberlerde bu adımlar ayrıca işaretlenmiştir.
- **Temizlik:** Her rehberin sonunda kendi temizlik bölümü vardır.

## Kullanım Kuralları

- **"✅ Gerçek çıktı"** satırları canlı test çalıştırmalarından birebir alınmıştır. Pod IP'leri, hostname'ler ve süreler lab ortamına aittir; sizinkiler farklı olacaktır.
- YAML dosyalarındaki **`REPLACE_ME_*`** yer tutucuları kendi ortamınıza göre doldurulmalıdır (namespace, CIDR, node adları, bucket adları...). Her rehber bunu nasıl yapacağınızı gösterir, genelde tek satırlık bir `sed` ile.
- Lab ağ adresleri `<LAN>`, `<UZAK>`, `<KURUM_AGI>` olarak kısaltılmıştır. Dosyalardaki şifreler (`PocDemo2026x`, `app-pass-2026`...) demo değerleridir; değiştirin.
- Rehberlerin oluşturduğu tüm namespace ve kullanıcılar `poc-` ya da `ocp-poc` ile başlar; bulmak ve temizlemek kolaydır.

## Diller

Her rehber aynı klasörde yan yana İngilizce (`README.md`) ve Türkçe (`README.tr.md`) olarak bulunur. YAML dosyaları ve içlerindeki yorumlar İngilizcedir ve iki dil için ortaktır. Düzeltme ve çeviri katkılarına açığız; issue ya da pull request açabilirsiniz.

## Sorumluluk Reddi

Bu bağımsız bir topluluk projesidir; Red Hat ile bağlantılı değildir ve Red Hat tarafından onaylanmamıştır. Rehberler, tek bir lab cluster'ında belirli bir zamanda yaşananları anlatır. Her adımı çalıştırmadan önce okuyun; cluster genelindeki değişiklikleri paylaşımlı ya da canlı bir cluster'da platform sahibinin onayı olmadan uygulamayın.

## Lisans

[Apache License 2.0](LICENSE)
