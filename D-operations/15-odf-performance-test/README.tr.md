# 15 — ODF Performans Testi — fio ile IOPS / Throughput / Latency

> 🇬🇧 [English](README.md) · [← 14 — MultiCluster Observability](../14-multicluster-observability/README.tr.md) · [POC akışı](../../README.tr.md) · [16 — OADP Backup/Restore →](../16-oadp/README.tr.md)

Bu doküman, ODF (OpenShift Data Foundation) üzerinde **fio** ile performans testinin **nasıl yapılacağını** anlatır. Komutlar, `ocs-storagecluster-ceph-rbd` (varsayılan, RWO block) storage class'ı ile sağlanan bir PVC üzerinde lab ortamında (OpenShift 4.22, ODF 4.22, Ceph `HEALTH_OK`) **canlı çalıştırılmış** ve sonuçları aşağıdaki bölümlere eklenmiştir. Sonuçlar ortamın donanımına (disk tipi, ağ hızı, node sayısı) bağlıdır; müşteri ortamında aynı testler tekrarlanmalıdır.

---

## 1. Test Ortamını Kurma

```bash
oc create ns poc-odf-perf-test

cat <<'EOF' | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fio-test-pvc
  namespace: poc-odf-perf-test
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: ocs-storagecluster-ceph-rbd   # test etmek istediğiniz storage class
  resources:
    requests:
      storage: 20Gi
EOF

oc run fio-bench -n poc-odf-perf-test --image=docker.io/ljishen/fio:latest \
  --overrides='{"spec":{"containers":[{"name":"fio","image":"docker.io/ljishen/fio:latest","command":["sh","-c","sleep 3600"],"volumeMounts":[{"name":"data","mountPath":"/data"}]}],"volumes":[{"name":"data","persistentVolumeClaim":{"claimName":"fio-test-pvc"}}]}}'

oc wait --for=condition=Ready pod/fio-bench -n poc-odf-perf-test --timeout=60s
```

`docker.io/ljishen/fio` — sadece `fio` yüklü, hazır kullanılabilir bir image (public, bu clusterda test edildi, ekstra kurulum gerekmez).

---

## 2. IOPS Testi (Rastgele Okuma/Yazma)

```bash
oc exec fio-bench -n poc-odf-perf-test -- fio --name=iops-test --directory=/data --size=2G \
  --rw=randrw --rwmixread=70 --bs=4k --ioengine=libaio --direct=1 --iodepth=32 --numjobs=4 \
  --runtime=30 --time_based --group_reporting
```

Çıktıda bakılacak alanlar:
- `read: IOPS=...` / `write: IOPS=...` — saniyedeki G/Ç işlem sayısı.
- `Disk stats: util=...` — **%90'ın altındaysa** disk doygunlaşmamış demektir, sonuç depolamanın gerçek tavanını yansıtmıyor olabilir; daha yüksek `--iodepth`/`--numjobs` ile tekrar deneyin.

✅ **Gerçek çıktı (4k random, %70 okuma, iodepth=32 × 4 job, 30 sn):**

| Tur | Okuma IOPS | Yazma IOPS | `util` |
|---|---|---|---|
| 1 | 16,1k (62,9 MiB/s) | 6.916 (27,0 MiB/s) | %99,7 |
| 2 | 7.949 (31,1 MiB/s) | 3.408 (13,3 MiB/s) | — |

İki tur arasındaki ~2 katlık fark, paylaşımlı cluster'daki diğer iş yüklerinin etkisidir (bkz. Bölüm 5).

---

## 3. Throughput Testi (Ardışık Okuma/Yazma, MB/s)

```bash
oc exec fio-bench -n poc-odf-perf-test -- fio --name=throughput-test --directory=/data --size=2G \
  --rw=rw --rwmixread=50 --bs=1M --ioengine=libaio --direct=1 --iodepth=16 --numjobs=2 \
  --runtime=30 --time_based --group_reporting
```

Çıktıda bakılacak alanlar:
- `READ: bw=...` / `WRITE: bw=...` (Run status bölümünde, MB/s cinsinden) — bant genişliği.
- IOPS testinden **farklı amaç**: burada büyük blok (`1M`) + yüksek `iodepth` bilinçli olarak throughput'u maksimize eder, bu da latency'yi yükseltir (bu normal — throughput ve latency aynı testte ikisi birden optimize edilemez).

✅ **Gerçek çıktı (1M sequential, %50 okuma, 30 sn):** Tur 1: `READ 54,6 MiB/s`, `WRITE 58,7 MiB/s` (`util=%100`). Tur 2: `READ 52,1 MiB/s`, `WRITE 55,7 MiB/s`. Toplam ~110 MiB/s, iki turda tutarlı. Bu değer büyük ihtimalle disklerden çok **node'lar arası ağ bant genişliğiyle** sınırlıdır (Ceph her yazmayı 3 kopya halinde ağ üzerinden replike eder); müşteri ortamında storage ağının hızı mutlaka sorulmalıdır.

---

## 4. Latency Testi (Saf G/Ç Gecikmesi)

```bash
oc exec fio-bench -n poc-odf-perf-test -- fio --name=latency-test --directory=/data --size=1G \
  --rw=randrw --rwmixread=70 --bs=4k --ioengine=libaio --direct=1 --iodepth=1 --numjobs=1 \
  --runtime=30 --time_based --group_reporting
```

`--iodepth=1` **kasıtlı** — kuyruklama olmadan tek bir G/Ç'nin gerçek round-trip süresini ölçer (IOPS testindeki `iodepth=32` ile latency ölçerseniz, kuyrukta bekleme süresi de gecikmeye karışır, yanıltıcı olur).

Çıktıda bakılacak alanlar:
- `clat (usec/msec): avg=...` — ortalama tamamlanma gecikmesi.
- `clat percentiles` satırındaki `50.00th` (medyan), `99.00th`, `99.99th` — **sadece ortalamaya bakmayın**, tail latency (p99/p99.9) gerçek kullanıcı deneyimini/SLA riskini ortalamadan çok daha iyi yansıtır.

✅ **Gerçek çıktı (4k random, iodepth=1, 30 sn):**

| | Okuma p50 | Okuma p99 | Yazma p50 | Yazma p99 | Ortalama (okuma / yazma) |
|---|---|---|---|---|---|
| Tur 1 | 0,66 ms | 9,4 ms | 1,9 ms | 46 ms | 1,19 ms / 4,13 ms |
| Tur 2 | 0,56 ms | 22 ms | 1,5 ms | 99 ms | — |

Medyan gecikme düşük ve tutarlı, ama **p99 turlar arasında 2 katın üzerinde** değişiyor ve p99.9'da 200 ms'yi aşan değerler var. Ortalama tek başına bu kuyruk gecikmelerini gizler; veritabanı gibi gecikmeye duyarlı iş yükleri için p99 değerleri esas alınmalıdır.

---

## 5. Dikkat Edilecek Noktalar

- **Cluster paylaşımlıysa** (bu ortamdaki gibi, birden fazla namespace/tenant varsa), ölçümler diğer tenant'ların eşzamanlı yükünden etkilenir — sonuçlar "izole laboratuvar" sayıları değildir. Kararlı/karşılaştırılabilir sonuç istiyorsanız testi düşük yük saatinde veya dedike bir test cluster'ında tekrarlayın.
- **Aynı testi birkaç kez çalıştırıp** sonuçları karşılaştırın — tek bir çalıştırma, o anki arka plan yüküne bağlı olarak yanıltıcı olabilir.
- **`--direct=1`** kullanmayı unutmayın — bu, OS page cache'ini bypass eder, gerçekten diske gidildiğini garanti eder (aksi halde yüksek "IOPS" ölçüp aslında RAM'i test etmiş olabilirsiniz).
- Storage class'ı değiştirerek (`ocs-storagecluster-cephfs`, farklı bir storage class vb.) aynı testleri tekrarlayıp karşılaştırabilirsiniz.

---

## 6. Temizlik

```bash
oc delete namespace poc-odf-perf-test   # PVC ve verisi de silinir
```
