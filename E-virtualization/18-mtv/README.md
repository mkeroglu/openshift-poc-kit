# 18 — Migrating From VMware / OVA With MTV

> 🇹🇷 [Türkçe](README.tr.md) · [← 17.1 — Windows Boot Source](../17-virtualization/windows/README.md) · [POC flow](../../README.md) · [19 — NodeHealthCheck →](../19-node-health-check/README.md)

This guide covers migrating VMs from an existing virtualization platform (VMware vSphere, OVA, RHV, OpenStack) to OpenShift Virtualization with the **Migration Toolkit for Virtualization (MTV)**: provider definition, inventory, network/storage maps, plan and migration.

The tests were done on a lab cluster (OpenShift 4.22, OpenShift Virtualization 4.22.9, **MTV 2.12.9**) with the **OVA provider**; live vCenter VMs were not touched. Every step up to the plan and the maps worked, but **with this MTV version the OVA migration failed at the disk stage**. The root cause is explained below with evidence (section 7). The vSphere flow uses the same resource types; the differences are in sections 2 and 8.

Scenario order:

1. Concepts and prerequisites
2. Provider definition (vSphere and OVA)
3. Inventory: the VM, network and disk IDs to migrate
4. NetworkMap and StorageMap
5. Plan
6. Migration (starting it) and monitoring
7. Findings from the live test (OVA + MTV 2.12.9)
8. vSphere-specific notes (warm migration, VDDK)
9. Cleanup

---

## 1. Concepts and Prerequisites

| Resource | Role |
|---|---|
| `Provider` | Connection to the source (vSphere/OVA/RHV/OpenStack) and target (`host` = this cluster) platform |
| `NetworkMap` | The target counterpart of each source network (pod network or a Multus NAD) |
| `StorageMap` | The target storage class for each source datastore / disk |
| `Plan` | Which VMs, with which maps, into which namespace |
| `Migration` | One run of a plan |

During the migration, MTV copies the VM disk to a target PVC and adapts the guest operating system to KVM with **virt-v2v** (virtio drivers, guest agent, boot settings). It then creates a `VirtualMachine` with the same CPU/memory/firmware.

```bash
oc get csv -n openshift-mtv | grep mtv-operator             # MTV operator
oc get forkliftcontroller -n openshift-mtv                   # if missing: Installed Operators -> MTV -> create a ForkliftController
oc get providers.forklift.konveyor.io -n openshift-mtv       # the "host" (target) provider comes automatically
```

> Short-name collision: if operators such as ACM are installed, `oc get plan` / `oc get provider` may return other resources. Always use the full name: `plans.forklift.konveyor.io`, `providers.forklift.konveyor.io`.

✅ **Actual output:** `mtv-operator.v2.12.9 Succeeded`; providers: `host` (openshift, `Ready`), a vSphere provider (`Ready`), `ova` (`Ready`).

**Console:** the **Migration** menu (appears in the left menu once MTV is installed) → **Providers / Plans / NetworkMaps / StorageMaps**.

---

## 2. Provider Definition

### 2.1 vSphere

```bash
oc create secret generic vsphere-credentials -n openshift-mtv \
  --from-literal=user='<vcenter-user>' --from-literal=password='<password>' \
  --from-literal=url='https://<vcenter>/sdk' --from-literal=insecureSkipVerify=true
```

```yaml
apiVersion: forklift.konveyor.io/v1beta1
kind: Provider
metadata:
  name: vcenter
  namespace: openshift-mtv
spec:
  type: vsphere
  url: https://<vcenter>/sdk
  secret: {name: vsphere-credentials, namespace: openshift-mtv}
  settings:
    vddkInitImage: <registry>/vddk:<version>   # recommended: an init image built with the VMware VDDK library
    sdkEndpoint: vcenter                        # or esxi
```

- The user needs at least read-only + VM snapshot/disk read permissions (the "vSphere privileges" list in the MTV documentation).
- A **VDDK image** speeds up disk transfer from vSphere significantly; build it once from VMware's VDDK package and put it in an internal registry.

### 2.2 OVA (NFS share)

The OVA provider scans an **NFS share** that holds `.ova` files (or unpacked `.ovf` + disk directories):

```yaml
apiVersion: forklift.konveyor.io/v1beta1
kind: Provider
metadata:
  name: ova
  namespace: openshift-mtv
spec:
  type: ova
  url: <nfs-server>:/ova
```

✅ **Actual output:** OVA provider `Ready`. The log of the OVA server pod (`ova-<id>`) showed a `Processing OVA file: /ova/<file>.ova` line for each file in the share, and `Processing OVF file: /ova/<dir>/<name>.ovf` for an unpacked directory.

**Console:** **Migration → Providers for virtualization → Create Provider** → choose the type (VMware / Open Virtual Appliance) → connection details.

---

## 3. Inventory: VM, Network and Disk IDs

Plans and maps refer to source objects by **ID**. The IDs come from the MTV inventory API (the Console lets you pick them from a list):

```bash
INV=$(oc get route forklift-inventory -n openshift-mtv -o jsonpath='{.spec.host}')
TOKEN=$(oc create token forklift-controller -n openshift-mtv --duration=10m)
PID=$(oc get providers.forklift.konveyor.io ova -n openshift-mtv -o jsonpath='{.metadata.uid}')

curl -sk -H "Authorization: Bearer $TOKEN" "https://$INV/providers/ova/$PID/vms?detail=1"   # VMs, disks, networks
curl -sk -H "Authorization: Bearer $TOKEN" "https://$INV/providers/ova/$PID/networks"
curl -sk -H "Authorization: Bearer $TOKEN" "https://$INV/providers/ova/$PID/storages"
# for vSphere the paths are: /providers/vsphere/<uid>/vms , /networks , /datastores
```

✅ **Actual output (a RHEL CoreOS OVA exported from VMware):** `name: "RHEL CoreOS 9.6"`, `cpuCount: 2`, `memoryMB: 4096`, `firmware: efi`, `osType: rhel8_64Guest`, disk `disk.vmdk` (16 GiB), NIC `Network adapter 1` → network `VM Network`. Concern: **`Invalid VM Name`** (the name has a space and uppercase letters; it cannot be a Kubernetes name).

> The `Invalid VM Name` warning is not blocking; a valid name is given in the Plan with `targetName` (section 5).

---

## 4. NetworkMap and StorageMap

`maps.yaml` (fill in the IDs from the inventory):

```bash
sed -e 's/REPLACE_ME_SOURCE_PROVIDER/ova/' \
    -e 's/REPLACE_ME_SOURCE_NETWORK_ID/<network-id>/' \
    -e 's/REPLACE_ME_SOURCE_STORAGE_ID/<disk-or-datastore-id>/' maps.yaml | oc apply -f -
```

- Network target `type: pod` → the VM is connected to the pod network. For a corporate VLAN use the NAD from section 9 of [17 — Virtualization](../17-virtualization/README.md): `type: multus, name: <namespace>/<nad-name>`.
- Disk target `ocs-storagecluster-ceph-rbd-virtualization` → an RWX Block disk suitable for live migration.

✅ **Actual output:** `poc-ova-network-map` and `poc-ova-storage-map` → `Ready=True`.

**Console:** **Migration → NetworkMaps / StorageMaps → Create** — sources and targets are picked from a list. The plan creation wizard can also generate the maps itself.

---

## 5. Plan

```bash
oc apply -f namespace.yaml        # target namespace: poc-mtv-demo
sed -e 's/REPLACE_ME_SOURCE_PROVIDER/ova/' -e 's/REPLACE_ME_SOURCE_VM_ID/<vm-id>/' \
    -e 's/REPLACE_ME_TARGET_VM_NAME/rhcos-from-ova/' plan.yaml | csplit -s - '/^---$/'   # xx00 = Plan, xx01 = Migration
oc apply -f xx00
oc get plans.forklift.konveyor.io poc-ova-plan -n openshift-mtv \
  -o jsonpath='{range .status.conditions[*]}{.type}={.status} {.message}{"\n"}{end}'
```

- `warm: false` → **cold migration**: the source VM is shut down and the disk is copied once. OVA only supports cold.
- `targetName`: required if the source name is not a valid Kubernetes name.
- When the plan is created, MTV runs pre-checks (are the maps complete, does the target namespace exist, does the VM have a blocking concern).

✅ **Actual output:** `Ready=True The migration plan is ready.`

**Console:** **Migration → Plans for virtualization → Create Plan** → source provider → select VMs → target namespace → maps → **Create migration plan**.

---

## 6. Migration and Monitoring

```bash
oc apply -f xx01      # Migration: poc-ova-migration-1
oc get migrations.forklift.konveyor.io poc-ova-migration-1 -n openshift-mtv \
  -o jsonpath='{range .status.vms[0].pipeline[*]}{.name} {.phase} {.error.reasons}{"\n"}{end}'
```

Expected pipeline (cold): `Initialize → DiskAllocation → ImageConversion (virt-v2v) → DiskTransferV2v → VirtualMachineCreation`. After a successful migration, a `VirtualMachine` is created in the target namespace (powered on or off depending on the plan's `targetPowerState`). Verify:

```bash
oc get vm,pvc -n poc-mtv-demo
virtctl start rhcos-from-ova -n poc-mtv-demo
virtctl console rhcos-from-ova -n poc-mtv-demo      # or virtctl vnc / the Console tab
```

**Console:** **Start** on the plan row → progress is shown step by step for each VM; click a step to open the pod logs.

---

## 7. Findings From the Live Test (OVA + MTV 2.12.9)

### 7.1 The OVA migration fails at the virt-v2v step

✅ **Actual output:**

```
Initialize      Completed   (10 s)
DiskAllocation  Completed   (22 s)
ImageConversion Running ->  "Guest conversion failed. See pod logs for details."
DiskTransferV2v Pending
VirtualMachineCreation Pending
```

Log of the conversion pod (`<plan>-<vm-id>-xxxxx`, in the target namespace):

```
Building command: virt-v2v [-v -x -o kubevirt -os /var/tmp/v2v -on rhcos-from-ova -i ova .]
ova: orig_ova = ., top_dir = /., ova_type = Directory
virt-v2v: error: /./root: Permission denied
```

The pod's environment variables showed `V2V_diskPath=.`; even though the OVA share was mounted in the pod at `/ova`, virt-v2v was given `.` (working directory `/`) instead of the OVA's path. In the inventory, the VM record's `ovfPath` field was empty. The same result came

- with the `.ova` archive,
- with the OVA unpacked into its own directory on NFS (`.ovf` + `.vmdk`; the OVA server said `Processing OVF file` and produced a new VM record),

both ways. In the same environment, an OVA migration done before the current MTV operator was installed (upgraded) was seen to have completed successfully. So the situation was assessed as a bug specific to the MTV build in use; no exactly matching entry was found in Red Hat's known issues list.

### 7.2 `skipGuestConversion: true` did not copy the disk

With virt-v2v skipped through `spec.skipGuestConversion: true` in the Plan, the migration `Succeeded` in **32 s** and a VM with the correct hardware was created on the target (2 vCPU, 4Gi, EFI, virtio disk, pod network, 16Gi PVC). But the pipeline had no disk transfer step, and when the VM started:

```
BdsDxe: failed to load Boot0001 "UEFI Misc Device" ...: Not Found
BdsDxe: No bootable option or device was found.
```

that is, **the disk was empty**. Don't use this option with an OVA source.

### 7.3 What to do in a POC

1. Before migrating, check the MTV version (`oc get csv -n openshift-mtv`) and look at that version's release notes / known issues.
2. **Do a trial migration with a single small test VM first**; in the `virt-v2v ... -i <type> <path>` command on the first lines of the conversion pod, confirm the path is filled in.
3. The real source is usually vSphere; OVA is only needed for exported images. The vSphere provider is not affected by this problem (it uses the vCenter/VDDK connection instead of a disk path).

---

## 8. vSphere-Specific Notes

- **Warm migration** (`warm: true`): while the VM keeps running, the disks are pre-copied with CBT (Changed Block Tracking); downtime is only the final delta + boot time. CBT must be enabled on the VM (`ctkEnabled=TRUE`). The `cutover` time is set in the Migration.
- **Windows VMs:** virt-v2v installs the virtio drivers automatically; Fast Startup / hibernation must be off on the source (otherwise the file system stays "dirty" and the conversion fails).
- **Static IP:** with `preserveStaticIPs: true`, MTV tries to keep the static IP settings of Windows/Linux guests; the target network (Multus NAD) must reach the same VLAN.
- **Network mapping:** if the VMs are on the same VLAN at the source, mapping them on the target to the VLAN NAD from 17 (`type: multus`) keeps the application's IP unchanged.

---

## 9. Cleanup

```bash
oc delete migrations.forklift.konveyor.io poc-ova-migration-1 -n openshift-mtv
oc delete plans.forklift.konveyor.io poc-ova-plan -n openshift-mtv
oc delete networkmaps.forklift.konveyor.io poc-ova-network-map -n openshift-mtv
oc delete storagemaps.forklift.konveyor.io poc-ova-storage-map -n openshift-mtv
oc delete project poc-mtv-demo
```

> After the plan is deleted, PVCs and conversion pods from failed attempts may remain in the target namespace; deleting the namespace cleans them up too.

---

## Summary

| Step | Status (lab, MTV 2.12.9, OVA) |
|---|---|
| Provider (`ova`, `host`) | ✅ `Ready` |
| Inventory (VM/network/disk IDs, concerns) | ✅ |
| NetworkMap / StorageMap | ✅ `Ready` |
| Plan (with `targetName`) | ✅ `Ready` |
| Migration — disk allocation | ✅ |
| Migration — virt-v2v conversion | ❌ `V2V_diskPath=.` (section 7.1) |
| Creating the VM with `skipGuestConversion` | ⚠️ VM created, disk empty (section 7.2) |
