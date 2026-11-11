# 15 — ODF Performance Test — IOPS / Throughput / Latency With fio

> 🇹🇷 [Türkçe](README.tr.md) · [← 14 — MultiCluster Observability](../14-multicluster-observability/README.md) · [POC flow](../../README.md) · [16 — OADP Backup/Restore →](../16-oadp/README.md)

This document explains **how to run** a performance test with **fio** on ODF (OpenShift Data Foundation). The commands were **run live on a lab cluster** (OpenShift 4.22, ODF 4.22, Ceph `HEALTH_OK`) against a PVC provisioned from the `ocs-storagecluster-ceph-rbd` (default, RWO block) storage class, and the results are included in the sections below. Results depend on the environment's hardware (disk type, network speed, number of nodes); repeat the same tests in your own environment.

---

## 1. Setting Up the Test Environment

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
  storageClassName: ocs-storagecluster-ceph-rbd   # the storage class you want to test
  resources:
    requests:
      storage: 20Gi
EOF

oc run fio-bench -n poc-odf-perf-test --image=docker.io/ljishen/fio:latest \
  --overrides='{"spec":{"containers":[{"name":"fio","image":"docker.io/ljishen/fio:latest","command":["sh","-c","sleep 3600"],"volumeMounts":[{"name":"data","mountPath":"/data"}]}],"volumes":[{"name":"data","persistentVolumeClaim":{"claimName":"fio-test-pvc"}}]}}'

oc wait --for=condition=Ready pod/fio-bench -n poc-odf-perf-test --timeout=60s
```

`docker.io/ljishen/fio` — a ready-to-use image with just `fio` installed (public, tested on this cluster, no extra setup needed).

---

## 2. IOPS Test (Random Read/Write)

```bash
oc exec fio-bench -n poc-odf-perf-test -- fio --name=iops-test --directory=/data --size=2G \
  --rw=randrw --rwmixread=70 --bs=4k --ioengine=libaio --direct=1 --iodepth=32 --numjobs=4 \
  --runtime=30 --time_based --group_reporting
```

Fields to look at in the output:
- `read: IOPS=...` / `write: IOPS=...` — I/O operations per second.
- `Disk stats: util=...` — **if below 90%**, the disk is not saturated and the result may not reflect the storage's real ceiling; try again with a higher `--iodepth`/`--numjobs`.

✅ **Actual output (4k random, 70% read, iodepth=32 × 4 jobs, 30 s):**

| Round | Read IOPS | Write IOPS | `util` |
|---|---|---|---|
| 1 | 16.1k (62.9 MiB/s) | 6,916 (27.0 MiB/s) | 99.7% |
| 2 | 7,949 (31.1 MiB/s) | 3,408 (13.3 MiB/s) | — |

The ~2x difference between the two rounds is the effect of other workloads on the shared cluster (see section 5).

---

## 3. Throughput Test (Sequential Read/Write, MB/s)

```bash
oc exec fio-bench -n poc-odf-perf-test -- fio --name=throughput-test --directory=/data --size=2G \
  --rw=rw --rwmixread=50 --bs=1M --ioengine=libaio --direct=1 --iodepth=16 --numjobs=2 \
  --runtime=30 --time_based --group_reporting
```

Fields to look at in the output:
- `READ: bw=...` / `WRITE: bw=...` (in the Run status section, in MB/s) — bandwidth.
- **A different goal from the IOPS test**: here a large block (`1M`) + high `iodepth` deliberately maximize throughput, which raises latency (this is normal — throughput and latency cannot both be optimized in the same test).

✅ **Actual output (1M sequential, 50% read, 30 s):** round 1: `READ 54.6 MiB/s`, `WRITE 58.7 MiB/s` (`util=100%`). Round 2: `READ 52.1 MiB/s`, `WRITE 55.7 MiB/s`. ~110 MiB/s total, consistent across both rounds. This value is most likely limited by the **network bandwidth between nodes** rather than the disks (Ceph replicates every write as 3 copies over the network); always ask about the storage network speed in your environment.

---

## 4. Latency Test (Pure I/O Latency)

```bash
oc exec fio-bench -n poc-odf-perf-test -- fio --name=latency-test --directory=/data --size=1G \
  --rw=randrw --rwmixread=70 --bs=4k --ioengine=libaio --direct=1 --iodepth=1 --numjobs=1 \
  --runtime=30 --time_based --group_reporting
```

`--iodepth=1` is **deliberate** — it measures the real round-trip time of a single I/O without queuing (if you measure latency with the IOPS test's `iodepth=32`, queue wait time gets mixed into the latency, which is misleading).

Fields to look at in the output:
- `clat (usec/msec): avg=...` — average completion latency.
- `50.00th` (median), `99.00th`, `99.99th` in the `clat percentiles` line — **don't look at the average only**; tail latency (p99/p99.9) reflects real user experience/SLA risk much better than the average.

✅ **Actual output (4k random, iodepth=1, 30 s):**

| | Read p50 | Read p99 | Write p50 | Write p99 | Average (read / write) |
|---|---|---|---|---|---|
| Round 1 | 0.66 ms | 9.4 ms | 1.9 ms | 46 ms | 1.19 ms / 4.13 ms |
| Round 2 | 0.56 ms | 22 ms | 1.5 ms | 99 ms | — |

Median latency is low and consistent, but **p99 varies more than 2x between rounds**, and there are values above 200 ms at p99.9. The average alone hides these tail latencies; for latency-sensitive workloads such as databases, use the p99 values.

---

## 5. Things to Keep in Mind

- **If the cluster is shared** (as in this environment, with several namespaces/tenants), measurements are affected by other tenants' concurrent load — the results are not "isolated lab" numbers. For stable/comparable results, repeat the test at low-load hours or on a dedicated test cluster.
- **Run the same test several times** and compare the results — a single run can be misleading depending on the background load at that moment.
- **Don't forget `--direct=1`** — it bypasses the OS page cache and guarantees the I/O really goes to disk (otherwise you may measure high "IOPS" while actually testing RAM).
- You can change the storage class (`ocs-storagecluster-cephfs`, another storage class, etc.), repeat the same tests and compare.

---

## 6. Cleanup

```bash
oc delete namespace poc-odf-perf-test   # the PVC and its data are deleted too
```
