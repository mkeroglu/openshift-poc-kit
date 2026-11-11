# 17.1 — Preparing a Boot Source (Golden Image) for Windows VMs

> 🇹🇷 [Türkçe](README.tr.md) · [← 17 — OpenShift Virtualization](../README.md) · [POC flow](../../../README.md) · [18 — Migration with MTV →](../../18-mtv/README.md)

When OpenShift Virtualization is installed, **Windows templates come with it** (Windows 10/11, Server 2016/2019/2022/2025). No extra configuration is needed for them. The difference from the RHEL templates is that **the operating system image (boot source)** the Windows templates use **is not shipped**. Because of Microsoft licensing, Red Hat cannot distribute a Windows image; the organization must provide it.

This document describes **four ways** to provide a Windows boot source. All four were **tested end to end on a lab cluster (OpenShift 4.22.6, OpenShift Virtualization 4.22.9, ODF 4.22.4) in the `ocp-poc-win` namespace, with the Windows Server 2022 Evaluation ISO**. At the end of each method, a VM was created from Red Hat's ready-made `windows2k22-server-medium` template, and it was confirmed that the VM booted as Windows Server 2022, the first-boot settings were applied, and RDP (3389) was reachable.

> **Test rounds:** all four methods were tested end to end in the first round. In the second round, after switching to the final naming, **method 3 (pipeline)** was run end to end again, and a VM was created from the resulting image with the ready-made template, re-verifying the first-boot settings and RDP (section 4). Methods 1, 2 and 4 were not rerun in the second round with the same manifests.

| # | Method | When | Live test result |
|---|---|---|---|
| 1 | Upload a ready image (`virtctl image-upload`) | When you have a sysprepped Windows disk (qcow2/raw) | ✅ 4.6 GB image uploaded in 5 min 15 s, VM created from the template |
| 2 | Install once from ISO and make a golden image | When you only have the ISO and the image is prepared once | ✅ Install + sysprep ~19 min, VM created from the template |
| 3 | The `windows-efi-installer` Tekton pipeline | Fully automated, repeatable image builds from the ISO | ✅ 31 min total, VM created from the template |
| 4 | Corporate registry + `DataImportCron` | Keeping the image up to date automatically, as with RHEL | ✅ Imported automatically; when a new version was pushed, the DataSource switched to the new image on its own; VM created from the template |

Contents:

0. Assessment: template exists, boot source doesn't
1. Common pieces: first boot (`unattend.xml`) and creating a VM from the template
2. Method 1 — Uploading a ready image
3. Method 2 — Install from ISO + golden image
4. Method 3 — Tekton pipeline (`windows-efi-installer`)
5. Method 4 — Corporate registry + `DataImportCron`
6. Making the boot source cluster-wide (`openshift-virtualization-os-images`)
7. Findings from the live test / things to watch
8. Cleanup

Files:

| File | Content |
|---|---|
| `namespace.yaml` | The `ocp-poc-win` namespace |
| `common/unattend-first-boot.yaml` | First-boot settings for VMs created from the golden image (skip OOBE, password, TZ, RDP) |
| `common/add-first-boot-sysprep.json` | JSON patch that attaches these settings to a VM created from the template |
| `1-upload/datasource.yaml` | DataSource that turns the uploaded image into a boot source |
| `2-iso/autounattend-configmap.yaml` | Unattended install + sysprep file for the ISO |
| `2-iso/vm-win2k22-installer.yaml` | Installer VM with ISO + blank disk + virtio drivers + sysprep |
| `2-iso/golden-image.yaml` | DataVolume + DataSource from the installed disk |
| `3-pipeline/pipelinerun-win2k22.yaml` | Run of the `windows-efi-installer` pipeline (adapted for OpenShift) |
| `4-registry-cron/Containerfile` | Packaging the Windows disk as a containerDisk |
| `4-registry-cron/dataimportcron.yaml` | Cron for automatic import from the registry |
| `4-registry-cron/test-registry.yaml` | **Test only**: a temporary registry backed by an ODF PVC |

---

## 0. Assessment: Template Exists, Boot Source Doesn't

```bash
# Windows templates
oc get template -n openshift -l template.kubevirt.io/type=base | grep -i windows

# Boot sources expected by the templates
oc get datasource -n openshift-virtualization-os-images \
  -o custom-columns=NAME:.metadata.name,READY:'.status.conditions[?(@.type=="Ready")].status',REASON:'.status.conditions[?(@.type=="Ready")].reason'

# Which boot sources are updated automatically
oc get dataimportcron -n openshift-virtualization-os-images
```

✅ **Actual output:**

- There are 24 Windows templates: `windows2k22-server-medium`, `windows2k22-server-large`, `windows2k22-highperformance-*`, `windows11-desktop-*`, etc.
- The `win10`, `win11`, `win2k16`, `win2k19`, `win2k22`, `win2k25` DataSources are **`Ready=False`, `NotFound`**. Each expects a PVC with its own name under `openshift-virtualization-os-images`, but there is no PVC.
- `rhel8`, `rhel9`, `fedora`, `centos-stream9`, etc. are **`Ready=True`**. They have `DataImportCron`s such as `rhel9-image-cron`, `fedora-image-cron`; their images are pulled automatically from the Red Hat registry. There is no such cron for Windows.

In the Console, the Windows template cards also show "Source not available" and cannot be quick-created directly.

```bash
oc process --parameters -n openshift windows2k22-server-medium
```

```
NAME                    VM name                           expression   windows2022-[a-z0-9]{6}
DATA_SOURCE_NAME        Name of the DataSource to clone                win2k22
DATA_SOURCE_NAMESPACE   Namespace of the DataSource                    openshift-virtualization-os-images
```

Because the template takes the boot source as a parameter, **we can point it at a golden image in our own namespace**. All the tests below were done this way, without touching the shared `openshift-virtualization-os-images`. To make it cluster-wide, see section 6.

---

## 1. Common Pieces

### 1.1 First-boot settings (`unattend.xml`)

Golden images are prepared with `sysprep /generalize /oobe`. Every VM created from such an image generates its own identity (computer name, SID) on first boot and lands on the OOBE screens (language, administrator password). `common/unattend-first-boot.yaml` skips these screens automatically:

- Computer name: random (`*`)
- Local `Administrator` password: `PocDemo2026x` (for the POC; in production give each VM its own ConfigMap/Secret)
- Time zone `Turkey Standard Time`, region `tr-TR`, **keyboard `en-US`** (see section 7) — change these to your own locale
- RDP enabled + RDP firewall rules on

It is attached to the VM as a **`sysprep` volume**:

```bash
oc apply -f namespace.yaml
oc apply -f common/unattend-first-boot.yaml
```

### 1.2 Creating a VM from the ready-made template (verification for every method)

```bash
# DATA_SOURCE_NAME: the DataSource the method produced (win2k22 / win2k22-uploaded / win2k22-from-iso / win2k22-registry)
oc process -n openshift windows2k22-server-medium \
  -p NAME=win-test -p DATA_SOURCE_NAME=win2k22 -p DATA_SOURCE_NAMESPACE=ocp-poc-win \
  | oc apply -n ocp-poc-win -f -

# The template creates the VM "Halted"; attach the first-boot settings before starting it
oc patch vm win-test -n ocp-poc-win --type=json --patch-file common/add-first-boot-sysprep.json
virtctl start win-test -n ocp-poc-win
```

**Verification:**

```bash
# OS, hostname, time zone through the guest agent
virtctl guestosinfo win-test -n ocp-poc-win

# RDP port (from inside the cluster)
IP=$(oc get vmi win-test -n ocp-poc-win -o jsonpath='{.status.interfaces[0].ipAddress}')
oc run rdp-check -n ocp-poc-win --rm -i --restart=Never --image=registry.access.redhat.com/ubi9/ubi-minimal \
  -- bash -c "timeout 5 bash -c '</dev/tcp/$IP/3389' && echo RDP OPEN"

# Graphical console
virtctl vnc win-test -n ocp-poc-win
```

To connect with RDP from outside: `virtctl port-forward vm/win-test 3389:3389 -n ocp-poc-win`, or a `Service` (NodePort/LoadBalancer), or a LAN IP via Multus (see `../README.md` section 9).

**Console:**

1. **Virtualization → Catalog → Template catalog** → "Microsoft Windows Server 2022 VM" (`windows2k22-server-medium`).
2. **Customize VirtualMachine** → **Disk source**: pick your own golden image as **"PVC (clone PVC)"** or as a **Bootable volume**. If the boot source has been made cluster-wide (section 6), this step is not needed; the card shows "Source available" directly.
3. **Scripts → Sysprep** → **Edit** → paste the XML from `common/unattend-first-boot.yaml` into the `unattend.xml` field.
4. **Create VirtualMachine**.

---

## 2. Method 1 — Uploading a Ready Image

**When:** the organization already has a prepared, sysprepped Windows disk. Examples: a golden image exported from another virtualization platform (VMware/Hyper-V), or an image produced once with method 2/3 and kept.

In the test, the golden disk produced by method 3 was first exported with `virtctl vmexport`, realistically creating the "we have a ready Windows disk" situation:

```bash
virtctl vmexport download win2k22-export -n ocp-poc-win --pvc=win2k22 \
  --output=win2k22-golden.img.gz --insecure
```

✅ **Actual output:** the 20 GB disk was downloaded as compressed raw (`.img.gz`, 4.6 GB) in **8 min 45 s**.

**Upload:**

```bash
virtctl image-upload dv win2k22-uploaded -n ocp-poc-win \
  --size=25Gi --image-path=./win2k22-golden.img.gz \
  --storage-class=ocs-storagecluster-ceph-rbd-virtualization \
  --access-mode=ReadWriteMany --volume-mode=block --insecure --force-bind

oc apply -f 1-upload/datasource.yaml        # DataSource win2k22-uploaded
```

> CDI accepts `qcow2`, `raw`, `vmdk`, `vhd(x)` and their `.gz`/`.xz` compressed forms, and converts them to raw. A `.vmdk` from VMware can be uploaded directly. To move a whole VMware VM, **Migration Toolkit for Virtualization (MTV)** is a better fit (see [18 — Migration with MTV](../../18-mtv/README.md)).

✅ **Actual output:** the upload took **5 min 15 s**, `DataSource win2k22-uploaded` → `Ready=True`. `win-from-upload`, created from the template:

```
hostname: ADMINAC-BDC0DA2 | timezone: Turkey Standard Time, 10800 | Windows Server 2022 Datacenter Evaluation
RDP 3389 OPEN -> 10.129.2.110 (393 s after the VM was started)
```

**Console:** **Virtualization → Bootable volumes → Add volume** → **Source type: Upload volume** → image file → **Volume name** `win2k22-uploaded`, **Preference** `windows.2k22.virtio` → **Save**.

---

## 3. Method 2 — Install From ISO + Golden Image

**When:** you only have the Microsoft ISO and the image is to be prepared once, in a controlled way. The flow is the Windows version of the flow in sections 4–5 of [17 — OpenShift Virtualization](../README.md) (ISO → install → golden image).

### 3.1 Uploading the ISO

```bash
# Windows Server 2022 Evaluation (180 days): https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2022
curl -L -o win2k22-eval.iso 'https://go.microsoft.com/fwlink/p/?LinkID=2195280&clcid=0x409&culture=en-us&country=US'

virtctl image-upload dv win2k22-iso-upload -n ocp-poc-win --size=6Gi \
  --image-path=./win2k22-eval.iso \
  --storage-class=ocs-storagecluster-ceph-rbd-virtualization --insecure --force-bind
```

✅ **Actual output:** the ISO is 5,044,094,976 bytes (`SSS_X64FREE_EN-US_DV9`, bootable). The upload took **7 min 25 s**.

### 3.2 The installer VM

`2-iso/vm-win2k22-installer.yaml`:

| Disk | Source | Role |
|---|---|---|
| `installcdrom` (sata, bootOrder 2) | `win2k22-iso-upload` PVC | Windows installation ISO |
| `rootdisk` (virtio, bootOrder 1) | 60Gi blank DataVolume (RWX Block) | Windows is installed here |
| `virtiocontainerdisk` (sata cdrom) | `registry.redhat.io/container-native-virtualization/virtio-win-rhel9` | virtio disk/network drivers, guest agent (`E:\`) |
| `sysprep` (sata cdrom) | `poc-win2k22-autounattend` ConfigMap | Unattended install file (`F:\`) |

- Instancetype `u1.large` (2 vCPU / 8Gi), preference `windows.2k22.virtio` (EFI + SecureBoot + TPM, Hyper-V enlightenments).
- The virtio-win image comes from the cluster's own definition: `oc get cm virtio-win -n openshift-cnv -o jsonpath='{.data.virtio-win-image}'`.

`2-iso/autounattend-configmap.yaml` is KubeVirt's upstream `windows2k22-autounattend` file (with `AcceptEula=true`). What it does:

1. Partitions the disk in an EFI/GPT layout, loads the virtio drivers from `E:\amd64\2k22`, and installs the `Windows Server 2022 SERVERDATACENTER` image.
2. Boots into **Audit** mode and runs `post-install.ps1`: installs the virtio guest tools + qemu-guest-agent and ejects the CD.
3. Runs **`sysprep /generalize /oobe`** and shuts the VM down. The result is a disk ready to become a golden image.

```bash
oc apply -f 2-iso/autounattend-configmap.yaml
oc apply -f 2-iso/vm-win2k22-installer.yaml
virtctl vnc win2k22-installer -n ocp-poc-win      # installer screen
```

> ⚠️ **"Press any key to boot from CD or DVD"**: in EFI mode, Windows ISOs show this prompt for about 5 s. If no key is pressed, it does not boot from the ISO. Press a key in the VNC console as soon as the VM starts.
>
> ✅ **Actual output:** the prompt was missed on the first boot: `BdsDxe: No bootable option or device was found.` After restarting the VM and pressing a key at boot, the **Windows Boot Manager → "Windows Setup [EMS Enabled]"** screen appeared. Because many keys were sent in a row in the test, the Boot Manager countdown also stopped and Enter had to be pressed. Pressing a single key in the console is enough. The pipeline in method 3 removes this prompt from the ISO, solving the problem entirely.

**For a manual (GUI) install:** remove the `sysprep` volume from the VM. Install from VNC; on the disk selection screen load the virtio disk driver with **"Load driver" → `E:\amd64\2k22`**. After installing, run `E:\virtio-win-guest-tools.exe`, and finally run:

```
C:\Windows\System32\Sysprep\sysprep.exe /generalize /oobe /shutdown
```

✅ **Actual output (unattended):** after the install started (~11:58), the VM shut itself down via sysprep (`12:17:01 Stopped`). Install + updates + sysprep took **~19 min**.

**Console:** **Catalog → Template catalog → windows2k22-server-medium** → **Customize VirtualMachine**:

- Tick **Boot from CD** → **CD source**: `win2k22-iso-upload` (or URL / Upload).
- **Disk source**: **Blank** (60Gi).
- Tick **"Mount Windows drivers disk"** (the virtio-win CD is attached automatically).
- **Scripts → Sysprep** → paste the XML from the ConfigMap into the `autounattend.xml` field. Leave it empty for a manual install.

### 3.3 Golden image

```bash
oc apply -f 2-iso/golden-image.yaml     # DataVolume + DataSource win2k22-from-iso
```

✅ **Actual output:** `win-from-iso`, created from the template:

```
hostname: ADMINAC-CHN6LTE | timezone: Turkey Standard Time, 10800 | Windows Server 2022 Datacenter Evaluation | agent: 110.2.2
RDP 3389 OPEN -> 10.128.2.148 (412 s after the VM was started)
```

(Guest agent 110.2.2 = the version from Red Hat's virtio-win image.)

**Console:** **Bootable volumes → Add volume → Use existing volume** → PVC `win2k22-installer-rootdisk`.

---

## 4. Method 3 — Tekton Pipeline (`windows-efi-installer`)

**When:** you want the Windows image to be built **fully automatically and repeatably**; e.g. building a new image from the latest ISO every month. The pipeline does everything from method 2 itself: downloads the ISO from Microsoft, removes the "press any key" prompt from the ISO, installs unattended, runs sysprep, creates the DataSource and cleans up the temporary resources.

**Prerequisites:**

- OpenShift Pipelines installed (in the lab test: `openshift-pipelines-operator-rh.v1.23.1`).
- Hub resolver enabled: `oc get cm resolvers-feature-flags -n openshift-pipelines` → `enable-hub-resolver: "true"`. The pipeline and tasks are pulled from the `kubevirt-tekton-pipelines` catalog on ArtifactHub; the cluster needs internet egress. On an air-gapped network, get the pipeline YAML from `https://github.com/kubevirt/kubevirt-tekton-tasks/releases`, apply it to the cluster and use the local name in `pipelineRef`.
- A `pipeline` ServiceAccount in the namespace (OpenShift Pipelines creates it automatically; it does not in `openshift-*` / `kube-*` namespaces).

> In 4.22 the SSP operator no longer ships the Tekton tasks/pipelines itself; that is why the pipeline is taken from ArtifactHub / GitHub releases.

```bash
oc create -f 3-pipeline/pipelinerun-win2k22.yaml
tkn pipelinerun logs -f -n ocp-poc-win        # or Console: Pipelines → PipelineRuns
```

> ⚠️ **OpenShift adaptation:** the `taskRunSpecs` block in the upstream example PipelineRun (`runAsUser: 107`, `fsGroup: 107`) is rejected by the SCC on OpenShift.
>
> ✅ **Actual output (first try):** `modify-windows-iso-file` → `PodAdmissionFailed: ... provider pipelines-scc: .spec.securityContext.fsGroup: Invalid value: [107]: 107 is not an allowed group`. With the block removed (as in the file), the second try had no problems.

✅ **Actual output (successful run):**

| Task | Start → End (UTC) | Duration |
|---|---|---|
| `import-autounattend-configmaps`, `import-win-iso`, `create-vm-root-disk` | 10:53:30 → 10:53:51 | ~20 s (the ISO was downloaded by CDI in the background) |
| `modify-windows-iso-file` (waiting for the ISO download + removing "press any key") | 10:53:54 → 11:08:24 | 14.5 min |
| `create-vm` + `wait-for-vmi-status` (unattended install + sysprep) | 11:08:24 → 11:24:40 | 16 min |
| `create-datasource-root-disk`, `cleanup-vm`, `delete-imported-iso`, `delete-imported-configmaps` | 11:24:41 → 11:25:01 | 20 s |
| **Total** | | **31 min**, `Succeeded` |

Result: `DataVolume win2k22` (20Gi, RWX Block) + `DataSource win2k22`. The temporary VM, ISO and ConfigMaps were deleted by the pipeline.

✅ **Second test round:** the pipeline `Succeeded` in **22 min** (ISO download + edit ~11 min, unattended install + sysprep ~10 min). VM created from the template: `Windows Server 2022 Datacenter Evaluation`, `timezone: Turkey Standard Time`, random hostname; **RDP 3389 open 305 s after the VM was started**.

`win-from-pipeline`, created from the template:

```
{"prettyName":"Windows Server 2022 Datacenter Evaluation","versionId":"2022","kernelRelease":"20348"}
hostname: ADMINAC-52MM6AG | timezone: Turkey Standard Time, 10800
RDP 3389 OPEN -> 10.129.2.81 (270 s after the VM was started)
```

On VNC the lock screen appeared directly without any OOBE screen (local date and time). Logged in with `Administrator` / `PocDemo2026x`. PowerShell: `fDenyTSConnections=0`, `TermService Running`.

**Parameters** (`3-pipeline/pipelinerun-win2k22.yaml`):

| Parameter | Value | Note |
|---|---|---|
| `winImageDownloadURL` | Server 2022 Evaluation link | For a licensed corporate ISO, give an internal web server URL |
| `acceptEula` | `true` | Acceptance of the Microsoft EULA |
| `preferenceName` | `windows.2k22.virtio` | |
| `autounattendConfigMapName` | `windows2k22-autounattend` | Windows 11: `windows11-autounattend`, 2025: `windows2k25-autounattend` |
| `baseDvName` | `win2k22` | Name of the produced golden image |

**Console:** apply the file with **Pipelines → PipelineRuns → Create PipelineRun** (YAML) or **Import YAML**. Progress is shown visually (task boxes).

---

## 5. Method 4 — Corporate Registry + `DataImportCron`

**When:** you want the Windows image to be **kept up to date automatically**, like RHEL/Fedora. The organization's image team pushes the new image (e.g. patched monthly) to the registry. The `DataImportCron` on the cluster polls the registry on a schedule; when it sees a new digest it imports the image and points the DataSource at the new image. Templates always create VMs from the latest image.

### 5.1 Building a containerDisk and pushing it to the registry

```bash
# Golden disk (from method 1/2/3) -> compressed qcow2 -> containerDisk
qemu-img convert -c -O qcow2 win2k22-golden.raw win2k22-golden.qcow2     # DON'T use .gz (see 5.2)
podman build -t <registry>/windows/win2k22:$(date +%Y%m%d) \
  --build-arg DISK=win2k22-golden.qcow2 -f 4-registry-cron/Containerfile .
podman push <registry>/windows/win2k22:$(date +%Y%m%d)
podman tag  <registry>/windows/win2k22:$(date +%Y%m%d) <registry>/windows/win2k22:latest
podman push <registry>/windows/win2k22:latest
```

The `Containerfile` puts the disk under `/disk/`, owned by the qemu user (UID 107). The disk must be **qcow2 or raw**; `.gz`/`.xz` is not unpacked from a registry source (see 5.2).

> **Test environment note:** the lab's internal image registry runs on `emptyDir`. A large image can fill the node disk, and the image is lost if the pod restarts. Quay was installed, but there was no test account. So for the test, a temporary OCI registry backed by an ODF PVC was set up in `ocp-poc-win` with `4-registry-cron/test-registry.yaml` to stand in for the corporate registry. In production, the organization's Quay/Harbor/Nexus is used.

✅ **Actual output:** the push (qcow2, 4.6 GiB) took **4 min 35 s**. In the registry: `{"name":"windows/win2k22","tags":["20260930-2","20260930","latest"]}`.

### 5.2 `DataImportCron`

```bash
# The registry's TLS CA — must exist in BOTH the target namespace AND openshift-cnv, with the SAME name (see the warning below)
for ns in ocp-poc-win openshift-cnv; do
  oc create configmap poc-registry-ca -n $ns --from-file=ca.crt=./registry-ca.crt
done

sed "s|REPLACE_ME_REGISTRY|<registry-host>|" 4-registry-cron/dataimportcron.yaml | oc apply -f -
```

- `schedule: "0 3 * * 1"`: checks for a new digest every Monday at 03:00. It also polls once right after creation.
- `managedDataSource: win2k22-registry`: creates this DataSource and always points it at the most recently imported image.
- `importsToKeep: 2`: the last 2 old images are kept; older ones are deleted.
- For an authenticated registry: `source.registry.secretRef` (a Secret with `accessKeyId` / `secretKey` fields).

> ⚠️ **`certConfigMap` is needed in two namespaces:** the job that polls the registry (`initial-job-<cron>-*`) runs in the `openshift-cnv` namespace and looks for the `certConfigMap` there. The pod that imports the image looks for the ConfigMap in the DataImportCron's own namespace.
>
> ✅ **Actual output (first try, ConfigMap only in `ocp-poc-win`):** cron `UpToDate=False NoDigest`. The polling pod stayed in `ContainerCreating`: `MountVolume.SetUp failed for volume "cdi-cert-vol" : configmap "registry-ca" not found` (`openshift-cnv`). Once a ConfigMap with the same name was created in `openshift-cnv` too, the polling job completed in **5 s** and the digest was read.

✅ **Actual output (first version):** the polling job read the digest (`sha256:9d11b5c2...`). The cron imported the `win2k22-registry-9d11b5c26657` DataVolume, whose name is derived from the digest, and `DataSource win2k22-registry` became ready. On ODF, CDI automatically turned the imported disk into a **VolumeSnapshot** (`DataSource src={"snapshot":{"name":"win2k22-registry-9d11b5c26657"}}`); RHEL boot sources are stored the same way.

> ⚠️ **Don't use `.gz` inside a containerDisk.** On the first push the disk was packaged as `.img.gz`. The import looked successful, but a VM created from this image could not boot: `BdsDxe: failed to load Boot0001 "UEFI QEMU HARDDISK" ... No bootable option or device was found`. The first bytes of the imported disk were `1f 8b 08 00` (a gzip header); that is, **CDI does not unpack the archive from a registry source, it writes the file to disk as-is**. From upload and HTTP sources it does unpack (method 1). The disk was converted to compressed qcow2 with `qemu-img convert -c -O qcow2` (20 GiB virtual, 4.6 GiB file) and pushed again.

### 5.3 Rolling out a new image version automatically

The image team publishing a new version was simulated: the qcow2 containerDisk was pushed to the same `latest` tag (new digest `sha256:e662cb06...`). So the cron would not wait for Monday 03:00, the poll was triggered by hand:

```bash
# The DataImportCron spec cannot be changed (webhook: "Cannot update DataImportCron Spec"), so
# the schedule cannot be shortened temporarily. To poll right away, create a Job from the CronJob:
CJ=$(oc get cronjob -n openshift-cnv -o name | grep win2k22-image-cron)
oc create job --from=$CJ manual-poll-1 -n openshift-cnv
```

✅ **Actual output:**

| Step | Result |
|---|---|
| Polling job | Completed in **7 s**, new digest detected |
| Import of the new version | `win2k22-registry-e662cb064934`, **284 s** |
| `DataSource win2k22-registry` | Switched to the new snapshot automatically (`src={"snapshot":{"name":"win2k22-registry-e662cb064934"}}`) |
| Old version | Kept because of `importsToKeep: 2` (`win2k22-registry-9d11b5c26657` snapshot `READY=true`) |
| Cron status | `UpToDate=True`, `lastImportedPVC=win2k22-registry-e662cb064934` |

`win-from-registry` (new version), created from the template:

```
hostname: ADMINAC-EVE4770 | timezone: Turkey Standard Time, 10800 | Windows Server 2022 Datacenter Evaluation
RDP 3389 OPEN -> 10.128.2.156 (319 s after the VM was started)
```

No change was made to the template or VM definitions; new VMs were created from the latest image automatically. Existing VMs are not affected, because they have their own disks.

### 5.4 Cluster-wide automatic Windows boot source (HCO)

The cron above is namespace-specific. To keep the Windows image up to date automatically **cluster-wide** (`openshift-virtualization-os-images/win2k22`), as with RHEL, the same definition is added to the HyperConverged resource. This makes the ready-made Windows templates "Source available" directly:

```yaml
# oc edit hco kubevirt-hyperconverged -n openshift-cnv
spec:
  dataImportCronTemplates:
    - metadata:
        name: win2k22-image-cron
        annotations:
          cdi.kubevirt.io/storage.bind.immediate.requested: "true"
      spec:
        schedule: "0 3 * * 1"
        managedDataSource: win2k22          # the DataSource name the templates expect
        importsToKeep: 2
        garbageCollect: Outdated
        template:
          spec:
            source:
              registry:
                url: docker://<corporate-registry>/windows/win2k22:latest
                certConfigMap: <registry-ca>   # in openshift-cnv and openshift-virtualization-os-images
            storage:
              resources:
                requests:
                  storage: 60Gi
```

> This step was **not applied** on the shared lab cluster (the HCO is cluster-wide). The same mechanism was tested live at the namespace level above.

---

## 6. Making the Boot Source Cluster-Wide

In the tests the golden images were kept in `ocp-poc-win` and passed to the template as a parameter. For the ready-made Windows templates to show **"Source available" directly in the Console catalog** and work without parameters, the image must be where the template expects it: the **`openshift-virtualization-os-images/win2k22`** PVC. There are three ways:

```bash
# a) Upload directly there, as in method 1
virtctl image-upload dv win2k22 -n openshift-virtualization-os-images --size=60Gi \
  --image-path=./win2k22-golden.img.gz --storage-class=ocs-storagecluster-ceph-rbd-virtualization --insecure

# b) Clone a prepared golden image there
cat <<'EOF' | oc apply -f -
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: win2k22
  namespace: openshift-virtualization-os-images
spec:
  source:
    pvc:
      name: win2k22
      namespace: ocp-poc-win
  storage:
    storageClassName: ocs-storagecluster-ceph-rbd-virtualization
    resources:
      requests:
        storage: 60Gi
EOF

# c) Run the method 3 pipeline in openshift-virtualization-os-images
#    or use HCO dataImportCronTemplates with method 4 (5.4)
```

Then `oc get datasource win2k22 -n openshift-virtualization-os-images` → `Ready=True`, and the `windows2k22-*` templates can be used directly.

> This step was **not applied** on the shared lab cluster. It affects the Windows templates of every cluster user, so do it with the platform owner's approval.

---

## 7. Findings From the Live Test / Things to Watch

- **The upstream pipeline example doesn't fit OpenShift as-is:** `taskRunSpecs` (`runAsUser/fsGroup: 107`) → `PodAdmissionFailed`. It worked once removed (section 4).
- **EFI "Press any key to boot from CD":** for a manual ISO install, press a key in the console as soon as the VM starts; otherwise `BdsDxe: No bootable option or device was found`. The pipeline removes this prompt from the ISO (section 3.2).
- **Keyboard layout and VNC:** on the first try the keyboard in `unattend.xml` was set to **Turkish Q** (`InputLocale 041f`). VNC/Console keys are sent as US-layout key codes, so the `-` character in the password went through as `*` and **logging in from the Console failed** (`The password is incorrect`). Setting the keyboard to `en-US` (`0409`) while leaving the region/time as `tr-TR` fixed it. If a local keyboard layout is needed, users can add it in their own session after logging in; users connecting over RDP use their own local keyboard.
- **RDP firewall rule:** in `unattend.xml`, `Networking-MPSSVC-Svc / FirewallGroups / <Group>Remote Desktop</Group>` **had no effect** on Server 2022. The RDP service was on (`fDenyTSConnections=0`), but the "Remote Desktop - User Mode (TCP-In)" rules were `Enabled=False` and 3389 was closed. Fixed with a `RunSynchronousCommand` in the `specialize` pass that runs `netsh advfirewall firewall set rule group="remote desktop" new enable=Yes` (`common/unattend-first-boot.yaml`).
- **Windows network profile `Public`:** on new VMs the network comes in the "Public" category, and the firewall accepts only explicitly allowed ports. Add rules for services other than RDP (WinRM 5985, SMB 445, etc.).
- **Golden image size:** the pipeline creates a 20Gi root disk. The templates request a 60Gi disk, so the cloned disk is 60Gi, but the `C:` partition inside Windows stays at 20 GB; it needs to be grown with `Resize-Partition`. For a permanent fix, increase the root disk size in the pipeline (the `create-vm-root-disk` manifest) or add a partition-extend command to the unattend file.
- **A Windows VM stuck in `Stopping`:** while on the UEFI/Boot Manager screen, the VM does not respond to the ACPI shutdown signal and stays in `Stopping` because of the long grace period in the Windows preference. Stop it with `virtctl stop <vm> --force --grace-period=0`.
- **`certConfigMap` in two namespaces:** see section 5.2.
- **No `.gz` in a containerDisk:** CDI does not unpack archives from a registry source; use qcow2/raw (section 5.2).
- **The `DataImportCron` spec cannot be changed:** to change the schedule/URL, delete and recreate the cron. For an immediate poll, use `oc create job --from=cronjob/...` (section 5.3).
- **License:** the test used the 180-day **Evaluation** edition. In production, use the organization's volume/KMS-licensed ISO and key. Activation can be left to the unattend file (`ProductKey`) or to KMS.

---

## 8. Cleanup

```bash
oc delete namespace ocp-poc-win
oc delete configmap poc-registry-ca -n openshift-cnv
oc delete job manual-poll-1 poc-manual-poll-1 -n openshift-cnv --ignore-not-found
```

(The `windows-efi-installer` pipeline already deletes its own temporary VM, ISO and ConfigMaps. Deleting the namespace also deletes the golden images, the test registry and the VMs.)
