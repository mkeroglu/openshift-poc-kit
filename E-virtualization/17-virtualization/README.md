# 17 — OpenShift Virtualization POC Guide

> 🇹🇷 [Türkçe](README.tr.md) · [← 16 — OADP Backup/Restore](../../D-operations/16-oadp/README.md) · [POC flow](../../README.md) · [17.1 — Windows Boot Source →](windows/README.md)

This document covers the virtual machine lifecycle with OpenShift Virtualization (KubeVirt) end to end: VMs from templates, installing from ISO, golden images, ODF storage, live migration, snapshot/restore, VLAN networking with Multus, and placement with affinity / anti-affinity / nodeSelector.

Every step is given for both the **CLI** (`oc` / `virtctl`) and the **Web Console**. All CLI steps were **tested end to end on a lab cluster (OpenShift 4.22, OpenShift Virtualization 4.22.9, ODF 4.22, bare-metal workers) in the `ocp-poc-virt` namespace** (two separate test rounds); the "✅ Actual output" lines come from those tests. The Console steps are written for the 4.22 UI.

Scenario order:

1. Prerequisite check
2. Operator installation
3. Templates — listing, exporting, VM from a template, custom template, template from an existing VM
4. VM installation from ISO (ISO upload + install + ejecting the CD-ROM)
5. Golden image (producing a boot source from an installed disk)
6. ODF storage — hotplug disk and online disk expansion
7. Live migration
8. Snapshot / restore
9. Multus — adding a VLAN (LAN) interface to a VM
10. Placement — nodeSelector, node affinity, VM affinity / anti-affinity
11. VM backup and restore with OADP
12. Known limitations / findings from the live test
13. Cleanup
14. Windows VMs → separate guide: [17.1 — Windows Boot Source](windows/README.md)

Files:

| File | Content |
|---|---|
| `namespace.yaml` | The `ocp-poc-virt` namespace |
| `operator-install.yaml`, `hyperconverged.yaml` | Operator installation (reference) |
| `custom-template.yaml` | A POC-specific template derived from `fedora-server-small` |
| `vm-to-template/vm-to-template.sh` | Script that produces a golden image + Template from an existing VM |
| `vm-to-template/poc-web-golden.yaml` | The template the script produced in the live test |
| `vm-from-iso.yaml` | VM with an ISO (CD-ROM) + blank disk |
| `golden-image.yaml` | DataVolume + DataSource + new VM from an installed disk |
| `data-disk-dv.yaml` | Data disk to hotplug |
| `live-migration.yaml` | `VirtualMachineInstanceMigration` |
| `vm-snapshot.yaml`, `vm-restore.yaml` | Snapshot and restore |
| `nad-vm-vlan.yaml` | `NetworkAttachmentDefinition` that connects VMs to a corporate VLAN (parametric) |
| `scheduling/*.yaml` | nodeSelector, node affinity, anti-affinity, affinity VMs |
| `oadp/namespace.yaml`, `oadp/cross-namespace-clone-rbac.yaml` | Backup test namespace and cross-namespace clone permission |
| `oadp/backup.yaml`, `oadp/restore.yaml` | Velero Backup / Restore |

---

## 1. Prerequisite Check

OpenShift Virtualization runs VMs with hardware virtualization (KVM) on the worker nodes.

```bash
# Which nodes can run VMs, do they have a KVM device?
oc get nodes -l kubevirt.io/schedulable=true \
  -o custom-columns=NODE:.metadata.name,KVM:'.status.allocatable.devices\.kubevirt\.io/kvm'
```

✅ **Actual output:** `KVM: 1k` on all 6 workers that can run VMs.

> **Important:** if the workers run as VMs on a hypervisor (e.g. VMware), **"Expose hardware assisted virtualization to the guest OS"** must be enabled for those VMs in ESXi. Otherwise there is no `/dev/kvm`, and VMs either don't start at all or run only with very slow software emulation (`spec.configuration.developerConfiguration.useEmulation`). That setting is for demos only; don't run performance tests with it.

On the storage side, ODF offers a separate storage class for VMs:

```bash
oc get sc ocs-storagecluster-ceph-rbd-virtualization
oc get storageprofile ocs-storagecluster-ceph-rbd-virtualization -o jsonpath='{.status.claimPropertySets}'
```

✅ **Actual output:** first preference `ReadWriteMany + Block`. Live migration needs RWX, so this storage class should be used.

---

## 2. Operator Installation

> In the lab the operator was already installed, so this step was not reapplied; only its status was verified. The YAMLs below match the real subscription on the cluster (`kubevirt-hyperconverged` / `stable` / `redhat-operators`).

**CLI:**

```bash
oc apply -f operator-install.yaml
oc get csv -n openshift-cnv -w          # wait until kubevirt-hyperconverged-operator ... Succeeded
oc apply -f hyperconverged.yaml
oc wait hco kubevirt-hyperconverged -n openshift-cnv --for=condition=Available --timeout=15m
```

The `virtctl` CLI can be downloaded from the cluster itself: in the Console **?** → **Command Line Tools** → **virtctl**, or from the `hyperconverged-cluster-cli-download-openshift-cnv.apps.<cluster-domain>` route.

**Console:**

1. **Operators → OperatorHub** → "OpenShift Virtualization" → **Install** (defaults: `openshift-cnv` namespace, `stable` channel).
2. When the install finishes, **Create HyperConverged** → **Create** with the defaults.
3. A **Virtualization** section appears in the left menu.

**Verification:**

```bash
oc get hco kubevirt-hyperconverged -n openshift-cnv \
  -o jsonpath='{range .status.conditions[*]}{.type}={.status}{"\n"}{end}'
```

✅ **Actual output:** `ReconcileComplete=True`, `Available=True`, `Progressing=False`, `Degraded=False`, `Upgradeable=True`.

Prepare the namespace and the SSH key to inject into the VMs:

```bash
oc apply -f namespace.yaml
ssh-keygen -t ed25519 -N '' -f ./vmkey
oc create secret generic vm-ssh-key -n ocp-poc-virt --from-file=key=./vmkey.pub
```

---

## 3. Templates

OpenShift Virtualization comes with ready-made VM templates in the `openshift` namespace (RHEL, CentOS Stream, Fedora, Windows ...). These templates take the operating system image from a **boot source** (`DataSource`). Boot sources are kept up to date automatically in the `openshift-virtualization-os-images` namespace.

### 3.1 Listing templates and boot sources

```bash
# Ready-made (base) templates
oc get template -n openshift -l template.kubevirt.io/type=base

# Boot sources (the OS images the templates clone)
oc get datasource -n openshift-virtualization-os-images
```

✅ **Actual output:** 77 base templates; 14 DataSources such as `fedora`, `centos-stream9`, `rhel9`, `rhel10`, `win2k22`. `fedora`, `rhel9` and `centos-stream9` are `Ready=True`.

### 3.2 Exporting a template and viewing its parameters

```bash
# Get the whole template as YAML
oc get template fedora-server-small -n openshift -o yaml > fedora-server-small-template.yaml

# See which parameters the template takes
oc process --parameters -n openshift fedora-server-small
```

✅ **Actual output:**

```
NAME                    DESCRIPTION                                          GENERATOR    VALUE
NAME                    VM name                                              expression   fedora-[a-z0-9]{16}
DATA_SOURCE_NAME        Name of the DataSource to clone                                   fedora
DATA_SOURCE_NAMESPACE   Namespace of the DataSource                                       openshift-virtualization-os-images
CLOUD_USER_PASSWORD     Randomized password for the cloud-init user fedora   expression   [a-z0-9]{4}-[a-z0-9]{4}-[a-z0-9]{4}
```

**Console:** **Virtualization → Templates** → select project `openshift` → click the template → **YAML** tab. **Download** at the top right saves it as a file.

### 3.3 Creating a VM from a template

**CLI:**

```bash
oc process -n openshift fedora-server-small \
  -p NAME=fedora-from-template -p CLOUD_USER_PASSWORD=Poc-Demo-2026 \
  | oc apply -n ocp-poc-virt -f -

# Add the SSH key to the VM (injected through cloud-init)
oc patch vm fedora-from-template -n ocp-poc-virt --type=merge -p \
  '{"spec":{"template":{"spec":{"accessCredentials":[{"sshPublicKey":{"source":{"secret":{"secretName":"vm-ssh-key"}},"propagationMethod":{"noCloud":{}}}}]}}}}'

virtctl start fedora-from-template -n ocp-poc-virt     # templates create the VM "Halted"
oc get dv,vm,vmi -n ocp-poc-virt
```

✅ **Actual output:** in both test rounds the VM was `Running` in **19–20 seconds** (DataVolume `Succeeded`). The disk was not downloaded from scratch; it was cloned from the boot source snapshot on ODF. Thanks to the storage profile, the disk was automatically created as `RWX Block` on `ocs-storagecluster-ceph-rbd-virtualization`.

**Access:**

```bash
virtctl ssh fedora@vmi/fedora-from-template -n ocp-poc-virt -i ./vmkey   # SSH (tunneled through the API)
virtctl console fedora-from-template -n ocp-poc-virt                      # serial console (exit: Ctrl+])
virtctl vnc fedora-from-template -n ocp-poc-virt                          # graphical console (needs a local VNC client)
```

✅ **Actual output:** `Fedora release 44`, 1 vCPU, 1.8Gi RAM, 30G `vda`, `qemu-guest-agent: active`.

**Console:**

1. **Virtualization → Catalog** → **Template catalog** tab.
2. Project: `ocp-poc-virt`; pick the "Fedora VM" (`fedora-server-small`) card.
3. When the boot source is ready, the card shows "Source available". Use **Customize VirtualMachine** to set the name, disk, cloud-init user/password and **SSH key** (pick the existing secret).
4. Create it with **Create VirtualMachine**. On the VM page, the **Console** tab gives the VNC/Serial console, and **Overview** shows the IP and node.

> **InstanceType instead of a template:** the newer recommended path in 4.x is the **InstanceTypes** tab. There you pick a boot volume, then a size (`u1.small` = 1 vCPU/2Gi, `u1.medium` ...). The VMs in section 10 are created this way (`spec.instancetype` + `spec.preference`).

### 3.4 Creating a custom (corporate) template

A ready-made template can be exported and modified into an organization-specific template. Templates placed in a project namespace appear in the Console catalog as **"User templates"** for that project.

`custom-template.yaml` is derived from `fedora-server-small`. The differences:

- Name `poc-fedora-small`, namespace `ocp-poc-virt`, display name "POC Fedora Small (SSH key + ODF virt)".
- Label `template.kubevirt.io/type: vm` (user template). The `app.kubernetes.io/*` labels that mark it as managed by the SSP operator were removed.
- The disk is always created on `ocs-storagecluster-ceph-rbd-virtualization`.
- A new **`SSH_KEY_SECRET`** parameter was added: the key is injected into the VM automatically.

```bash
oc apply -f custom-template.yaml
oc process --parameters poc-fedora-small -n ocp-poc-virt
oc process poc-fedora-small -n ocp-poc-virt -p NAME=poc-vm-from-custom-template \
  | oc apply -n ocp-poc-virt -f -
virtctl start poc-vm-from-custom-template -n ocp-poc-virt
virtctl ssh fedora@vmi/poc-vm-from-custom-template -n ocp-poc-virt -i ./vmkey
```

✅ **Actual output:** the parameter list showed `SSH_KEY_SECRET ... vm-ssh-key`. The disk was created on `ocs-storagecluster-ceph-rbd-virtualization`, and `virtctl ssh` logged in with the key without any extra patch.

**Console:**

- **Virtualization → Templates** → the template's **⋮** menu in the `openshift` project → **Clone**: give a new name and target project (`ocp-poc-virt`). Then customize the clone from its **YAML** / **Disks** / **Scheduling** tabs.
- To create one from scratch: **Templates → Create Template** (opens the YAML editor).

### 3.5 Creating a template from an existing (installed) VM

OpenShift Virtualization has no single-step "make a template from a VM" API (there is no `VirtualMachineTemplate`-like CRD on the cluster). It is done with this flow:

1. Customize the source VM (packages, settings, hardening).
2. **Generalize**: clean machine-specific identities and stop the VM.
3. Clone the VM's root disk and publish it as a **golden image** (`DataSource`).
4. Strip fields such as MAC/UUID from the VM definition and produce a **parameterized `Template`**. The disk is cloned from the golden image through `sourceRef`.

The `vm-to-template/vm-to-template.sh` script handles steps 3 and 4.

**1) Customization** (the `poc-vm-from-custom-template` VM was used in the test):

```bash
# inside the VM
sudo dnf install -y httpd
echo "<h1>POC golden web server</h1>" | sudo tee /var/www/html/index.html
sudo systemctl enable --now httpd
echo "POC corporate setting v1" | sudo tee /etc/poc-golden
```

✅ **Actual output:** `dnf_rc=0`, `curl localhost` → `<h1>POC golden web server</h1>`, `httpd enabled`.

**2) Generalize and stop:**

```bash
# inside the VM
sudo cloud-init clean --logs --seed      # so cloud-init runs again on new VMs (hostname, user, SSH key)
sudo truncate -s0 /etc/machine-id        # so each clone generates its own machine-id
sudo rm -f /etc/ssh/ssh_host_*           # so each clone generates its own SSH host keys
sudo rm -f /home/fedora/.ssh/authorized_keys
sudo hostnamectl set-hostname localhost
sudo systemctl poweroff
```

```bash
virtctl stop poc-vm-from-custom-template -n ocp-poc-virt
```

> On Windows VMs the equivalent of this step is `sysprep /generalize /oobe /shutdown`.

**3+4) Produce the golden image + Template:**

```bash
cd vm-to-template
./vm-to-template.sh poc-vm-from-custom-template ocp-poc-virt poc-web-golden "POC Web Server (httpd, golden)"
```

The script:

- Exits with an error if the VM is not stopped (to avoid an inconsistent image).
- Clones the root disk (lowest `bootOrder`) into the `poc-web-golden-image` DataVolume and publishes it as `DataSource poc-web-golden`.
- Removes the `firmware.uuid`, `firmware.serial` and `macAddress` fields, the `vm.kubevirt.io/template*` labels and hotplug/data disks from the VM definition.
- Produces a `Template` with `NAME`, `CLOUD_USER_PASSWORD` and `SSH_KEY_SECRET` parameters, saves it as `poc-web-golden.yaml` and applies it to the cluster.

✅ **Actual output:**

```
Root disk: volume=rootdisk pvc=poc-vm-from-custom-template size=30Gi sc=ocs-storagecluster-ceph-rbd-virtualization
datavolume.cdi.kubevirt.io/poc-web-golden-image created
datasource.cdi.kubevirt.io/poc-web-golden created
datavolume.cdi.kubevirt.io/poc-web-golden-image condition met
template.template.openshift.io/poc-web-golden created
```

The generated template (`vm-to-template/poc-web-golden.yaml`) was reviewed: no MAC/UUID, the disk is cloned with `sourceRef: DataSource ocp-poc-virt/poc-web-golden`, and CPU/memory/firmware/affinity are taken as-is from the source VM.

**5) New VMs from the template:**

```bash
oc process poc-web-golden -n ocp-poc-virt -p NAME=poc-web-01 | oc apply -n ocp-poc-virt -f -
oc process poc-web-golden -n ocp-poc-virt -p NAME=poc-web-02 | oc apply -n ocp-poc-virt -f -

# Verify: each VM should have a different hostname/machine-id, and httpd and /etc/poc-golden should be there
virtctl ssh fedora@vmi/poc-web-01 -n ocp-poc-virt -i ./vmkey \
  --command 'hostname; cat /etc/machine-id /etc/poc-golden; systemctl is-active httpd; curl -s localhost'
```

✅ **Actual output:** `oc process --parameters` → `NAME` (default `poc-web-golden-[a-z0-9]{6}`), `CLOUD_USER_PASSWORD`, `SSH_KEY_SECRET`. On the two VMs created from the template:

| | `poc-web-01` | `poc-web-02` |
|---|---|---|
| hostname | `poc-web-01` | `poc-web-02` |
| machine-id | `92a2346b...` | `de0affe7...` |
| machine-id (2nd round) | `298ee517...` | `f29cef4c...` |
| SSH host key | `SHA256:omHiiCKc...` | `SHA256:8bCZoXDD...` |
| MAC | `02:ed:88:63:8b:48` | `02:ed:88:63:8b:49` |
| `/etc/poc-golden` | `POC corporate setting v1` | `POC corporate setting v1` |
| httpd | `active`, `<h1>POC golden web server</h1>` | `active`, `<h1>POC golden web server</h1>` |

The customizations (package, service, settings file) came from the golden image. Machine-specific identities (hostname, machine-id, SSH host key, MAC) were regenerated on each VM; generalization worked correctly. The disks were created as 30Gi on `ocs-storagecluster-ceph-rbd-virtualization`.

> **Using the template in another namespace:** if a VM is created in another namespace while the template and golden image live in `ocp-poc-virt`, the disk clone waits with an `UnauthorizedDataVolumeCreate` error (see section 11.2). Clone permission must be granted in the source namespace: `oadp/cross-namespace-clone-rbac.yaml`.

**Console equivalent:**

1. Generalize and stop the VM (commands above, from the VM's **Console** tab).
2. **Virtualization → Bootable volumes → Add volume** → **Source type: Use existing volume** → PVC `poc-vm-from-custom-template` → name `poc-web-golden`. Or VM → **Snapshots → Take snapshot**, then **Add volume → Volume snapshot**.
3. **Virtualization → Templates** → a suitable base template (e.g. `fedora-server-small`) **⋮ → Clone** → project `ocp-poc-virt`. In the clone's **Disks** tab, choose the `poc-web-golden` volume as the boot source.
4. Create new VMs from **Catalog → Template catalog → User templates**.

> **If you only need an exact copy:** the `VirtualMachineClone` API (`clone.kubevirt.io`) copies a VM directly with its disk and definition, renewing MAC/SMBIOS values. Console: VM → **Actions → Clone**. This does not produce a template; it is for one-off copies.

---

## 4. VM Installation From ISO

Flow: upload the ISO to a PVC (DataVolume) → create the VM with the ISO as **CD-ROM** + a **blank disk** → boot from the ISO and install to disk → stop the VM and eject the CD-ROM → boot from disk.

For the test, the **Alpine Linux 3.24.2 "virt" ISO** (66 MB) was used because it is small and can be installed from the serial console. The flow is exactly the same for RHEL / Windows ISOs; only the installer screen differs (for Windows see section 12).

### 4.1 Uploading the ISO

**CLI** (`virtctl image-upload`, through the CDI upload proxy route):

```bash
curl -LO https://dl-cdn.alpinelinux.org/alpine/latest-stable/releases/x86_64/alpine-virt-3.24.2-x86_64.iso

virtctl image-upload dv alpine-iso -n ocp-poc-virt \
  --size=1Gi \
  --image-path=./alpine-virt-3.24.2-x86_64.iso \
  --storage-class=ocs-storagecluster-ceph-rbd-virtualization \
  --insecure --force-bind
```

✅ **Actual output:** `66.00 MiB ... 100.00%` (~35 MiB/s), `Processing completed successfully`. Total time **37 seconds**.

> `--insecure` is needed if the CDI upload proxy route's certificate is not trusted on the local machine. `--force-bind` makes the PVC bind right away on `WaitForFirstConsumer` storage classes.

If the ISO is on a web server, it can be pulled directly from a URL instead of being uploaded:

```yaml
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: alpine-iso
  namespace: ocp-poc-virt
spec:
  source:
    http:
      url: https://<web-server>/alpine-virt-3.24.2-x86_64.iso
  storage:
    resources:
      requests:
        storage: 1Gi
```

**Console:**

1. **Virtualization → Bootable volumes** → **Add volume**.
2. **Source type: Upload volume** → choose the ISO file (or give a URL with **Use URL**).
3. **Volume name** `alpine-iso`, **StorageClass** `ocs-storagecluster-ceph-rbd-virtualization`, **Disk size** `1Gi`.
4. Optionally pick a **Preference** (e.g. `alpine`), then **Save**. Upload progress is shown on screen.

### 4.2 Creating the VM that boots from the ISO

`vm-from-iso.yaml` defines two disks:

- `installation-cdrom`: the ISO PVC, `cdrom` / `sata`, **`bootOrder: 1`**
- `rootdisk`: a 5Gi **blank** (`source.blank`) DataVolume, RWX Block, **`bootOrder: 2`**

```bash
oc apply -f vm-from-iso.yaml
virtctl console alpine-from-iso -n ocp-poc-virt     # installer screen
```

**Console:**

1. **Virtualization → Catalog** → **Template catalog** → pick the template matching the OS (e.g. "Fedora VM" or "RHEL 9 VM").
2. **Customize VirtualMachine** → **Disk source**: tick **"Boot from CD"** → **CD source**: the `alpine-iso` PVC you uploaded. **Disk source**: **Blank** (e.g. 5Gi).
3. **Create VirtualMachine** → connect to the installer from the **Console** tab.

### 4.3 Installing the operating system to disk

Log in as `root` on the console and run the Alpine setup wizard. In the test the same steps were automated over the serial console with an answer file:

```sh
# On the VM console (live system booted from the ISO)
cat > /tmp/answers <<'EOF'
KEYMAPOPTS=none
HOSTNAMEOPTS=alpine-from-iso
INTERFACESOPTS="auto lo
iface lo inet loopback

auto eth0
iface eth0 inet dhcp
"
TIMEZONEOPTS=UTC
PROXYOPTS=none
APKREPOSOPTS="-1"
USEROPTS=none
SSHDOPTS=openssh
ROOTSSHKEY="<contents of vmkey.pub>"
NTPOPTS=none
DISKOPTS="-m sys /dev/vda"
LBUOPTS=none
APKCACHEOPTS=none
EOF
ERASE_DISKS=/dev/vda setup-alpine -e -f /tmp/answers
poweroff
```

✅ **Actual output:** packages were downloaded from the internet (over the pod network), `Installation is complete. Please reboot.`, `SETUP_RC=0`. In the second round the ISO upload took 17 s and the install 24 s.

### 4.4 Ejecting the CD-ROM and booting from disk

> On a VM with `runStrategy: Always`, a `poweroff` from inside the guest makes KubeVirt **restart** the VM (booting from the ISO again). So stop the VM with `virtctl stop`.

**CLI:**

```bash
virtctl stop alpine-from-iso -n ocp-poc-virt
oc wait vmi alpine-from-iso -n ocp-poc-virt --for=delete --timeout=120s

# Remove the CD-ROM disk and its volume from the VM definition (index 1 = installation-cdrom)
oc patch vm alpine-from-iso -n ocp-poc-virt --type=json -p \
  '[{"op":"remove","path":"/spec/template/spec/domain/devices/disks/1"},
    {"op":"remove","path":"/spec/template/spec/volumes/1"}]'

virtctl start alpine-from-iso -n ocp-poc-virt
virtctl ssh root@vmi/alpine-from-iso -n ocp-poc-virt -i ./vmkey
```

✅ **Actual output:** disks: `[{"bootOrder":2,"disk":{"bus":"virtio"},"name":"rootdisk"}]`. Over SSH: `alpine-from-iso`, `3.24.2`, `/dev/vda3  3.3G  60.8M  3.1G  2% /`. The system now boots from the installed disk.

**Console:** VM → **Configuration → Storage** (or **Disks**) → on the `installation-cdrom` row **⋮ → Eject CD-ROM** / **Detach** → **Actions → Restart**.

---

## 5. Golden Image (boot source from an installed disk)

A disk installed from ISO and customized can be turned into a "golden image". The image is added to the catalog as a boot source, and new VMs are cloned from it in seconds. This is how organizations distribute their own hardened RHEL/Windows images.

`golden-image.yaml` contains three resources:

1. `DataVolume alpine-golden`: a clone of the installed `alpine-from-iso-rootdisk`
2. `DataSource alpine-golden`: publishes that PVC as a boot source (with default instancetype/preference labels)
3. `VirtualMachine alpine-from-golden`: a new VM booted from that DataSource through `sourceRef`

```bash
virtctl stop alpine-from-iso -n ocp-poc-virt       # stop the source VM for a consistent copy
oc apply -f golden-image.yaml
virtctl start alpine-from-iso -n ocp-poc-virt
virtctl ssh root@vmi/alpine-from-golden -n ocp-poc-virt -i ./vmkey
```

✅ **Actual output:** `alpine-golden` and `alpine-from-golden-rootdisk` `Succeeded` in **~24 seconds**. The new VM booted: `golden image VM OK: hostname=alpine-from-iso alpine=3.24.2`.

> The hostname being `alpine-from-iso` is expected: the disk is copied exactly. For real golden images, **generalize** before taking the image: on Linux `virt-sysprep` / `cloud-init clean` + machine-id cleanup, on Windows `sysprep /generalize`.

**Console:** **Virtualization → Bootable volumes → Add volume → Source type: Use existing volume** (PVC `alpine-from-iso-rootdisk`) or **Volume snapshot**. Optionally mark the volume **"Set as default boot source"**. The new volume becomes selectable on the **Catalog → InstanceTypes** screen.

---

## 6. ODF Storage — Hotplug Disk and Online Expansion

### 6.1 Attaching a disk to a running VM (hotplug)

```bash
oc apply -f data-disk-dv.yaml                  # 10Gi blank, RWX Block DataVolume
oc wait dv fedora-data-disk -n ocp-poc-virt --for=condition=Ready

virtctl addvolume fedora-from-template -n ocp-poc-virt \
  --volume-name=fedora-data-disk --serial=DATADISK01 --persist
```

`--persist` also writes the disk into the VM definition, so the disk stays when the VM restarts. Without it, the disk is attached only to the running instance.

```bash
# inside the VM
lsblk -d -o NAME,SIZE,SERIAL
sudo mkfs.xfs /dev/sda && sudo mkdir -p /data && sudo mount /dev/sda /data
```

✅ **Actual output:** `sda  10G  datadisk01` appeared without shutting down the VM. Volume status `fedora-data-disk Ready` (with an `hp-volume-*` attach pod).

> Hotplug disks show up inside the VM on the **SCSI bus** (`/dev/sda`), not `virtio`. The `--serial` value appears lowercased. The disk is not under `/dev/disk/by-id/virtio-*`; use the UUID for persistent mounts.

**Console:** VM → **Configuration → Storage** → **Add disk** → **Use existing** / **Blank** → **Save** (added while the VM runs).

### 6.2 Expanding the disk online

```bash
oc patch pvc fedora-data-disk -n ocp-poc-virt --type=merge \
  -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'

# inside the VM (XFS)
sudo xfs_growfs -d /data
```

✅ **Actual output:** the PVC became `20Gi`. Inside the VM, without a restart, `sda 20G`. After `xfs_growfs`, `/dev/sda  20G ... /data`, and the test file was kept.

**Console:** **Storage → PersistentVolumeClaims** → `fedora-data-disk` → **Actions → Expand PVC**.

---

## 7. Live Migration

Moves a running VM to another node without shutting it down. During node maintenance (`oc adm drain`) VMs are moved this way automatically. Requirement: the VM disks must be **RWX** (here ODF RBD Block RWX).

```bash
oc get vmi -n ocp-poc-virt \
  -o custom-columns=VM:.metadata.name,NODE:.status.nodeName,MIGRATABLE:'.status.conditions[?(@.type=="LiveMigratable")].status'
```

**To measure the interruption**, a ping at 200 ms intervals and a timestamp counter were started inside the VM:

```bash
# inside the VM
GW=$(ip route | awk '/default/{print $3}')     # the pod network's gateway
nohup ping -i 0.2 -D $GW > /tmp/ping.log 2>&1 &
nohup sh -c 'while true; do date +%T.%N >> /tmp/counter.log; sleep 0.2; done' >/dev/null 2>&1 &
```

**CLI:**

```bash
oc apply -f live-migration.yaml          # or: virtctl migrate fedora-from-template -n ocp-poc-virt
oc get vmim -n ocp-poc-virt -w
oc get vmi fedora-from-template -n ocp-poc-virt -o jsonpath='{.status.migrationState}'
```

✅ **Actual output:**

| Measurement | Round 1 | Round 2 |
|---|---|---|
| Source → target | worker-A → worker-B | worker-A → worker-B |
| Mode | `PreCopy` | `PreCopy` |
| Duration | **7 s** | **9 s** |
| In-VM ping (0.2 s interval) | 209 packets, **0 lost** | 246 packets, **0 lost** |
| Largest ping / counter gap (switchover) | 0.58 s | 0.38 / 0.42 s |
| VM boot time (`uptime -s`) | Unchanged | Unchanged |

**Console:** VM → **Actions → Migrate → Compute** (or **⋮ → Migrate** in the VM list). Progress is tracked under **Virtualization → Overview → Migrations**.

---

## 8. Snapshot / Restore

Works on top of ODF RBD CSI snapshots. Snapshots can be taken while the VM is **running**. If the VM has `qemu-guest-agent`, the file system is frozen before the snapshot, giving an application-consistent copy.

```bash
# inside the VM: data that should be in the snapshot
echo "data before snapshot" > ~/important-file.txt; sync

oc apply -f vm-snapshot.yaml
oc wait vmsnapshot fedora-snap-1 -n ocp-poc-virt --for=condition=Ready
oc get vmsnapshot fedora-snap-1 -n ocp-poc-virt -o jsonpath='{.status.indications}'
```

✅ **Actual output:** `phase=Succeeded indications=["GuestAgent","Online"]`. Behind the scenes a 30Gi `VolumeSnapshot` was created with `ocs-storagecluster-rbdplugin-snapclass`.

**Disaster simulation and rollback:**

```bash
# inside the VM: "break" the data
rm -f ~/important-file.txt; echo mistake > ~/after-snapshot.txt

virtctl stop fedora-from-template -n ocp-poc-virt          # the VM must be off for restore
oc apply -f vm-restore.yaml
oc wait vmrestore fedora-restore-1 -n ocp-poc-virt --for=condition=Ready
virtctl start fedora-from-template -n ocp-poc-virt
```

✅ **Actual output:** before the restore, `after-snapshot.txt` was in the home directory. After the restore only `important-file.txt` was left, with the content `data before snapshot`. The file created later disappeared, and the deleted file came back.

> ⚠️ **Hotplug disk + SELinux: the snapshot may fail.** In the second test round a hotplug disk had been attached to the VM as in 6.1 and mounted at `/data`. The snapshot became `Failed` after 5 minutes:
>
> ```
> command Freeze failed: "LibvirtError(Code=113, ... guest agent command failed: unable to execute QEMU agent command
> 'guest-fsfreeze-freeze': failed to open /data: Permission denied')"
> ```
>
> **Cause:** on Fedora/RHEL, `qemu-guest-agent` runs in a restricted SELinux context (`virt_qemu_ga_t`). The root directory of the newly created XFS stays unlabeled (`unlabeled_t`), so the agent cannot open the mount point and cannot freeze the file system.
>
> **Fix:** label the mount point after mounting the disk (also applies before adding it to `/etc/fstab` for a persistent mount):
>
> ```bash
> sudo restorecon -v /data      # Relabeled /data from ...:unlabeled_t:s0 to ...:default_t:s0
> ```
>
> ✅ **Actual output:** after `restorecon` the snapshot `Succeeded` in **3 s**, and this time two VolumeSnapshots were taken, for the **root disk + the hotplug data disk**. After the restore (**2 s**), both the file in the home directory and `/data/test.txt` went back to their state at snapshot time.
>
> If a snapshot fails, a `VirtualMachineRestore` that depends on it never completes either and leaves the VM off; first confirm with `oc get vmsnapshot` that it is `Succeeded`.

**Console:** VM → **Snapshots** tab → **Take snapshot**. To roll back, stop the VM → the snapshot's **⋮ → Restore VirtualMachine from snapshot**. A **separate new VM** can also be created from a snapshot: **⋮ → Create VirtualMachine**.

---

## 9. Multus — VLAN (LAN) Interface for a VM

In addition to the pod network, a NIC from a corporate VLAN is added to the VM. The VM then becomes reachable by IP directly, like other servers on the LAN.

| Parameter | Meaning | In the lab test |
|---|---|---|
| `REPLACE_ME_BRIDGE` | Linux bridge created on the nodes with NNCP | `br-vm` |
| `REPLACE_ME_VLAN_ID` | VLAN the VMs connect to | the corporate server VLAN (has DHCP) |
| Corporate network (for the route) | Network(s) where the clients that will reach the VM live | `<CORP_NET>/16` |

### 9.1 Node side (NNCP)

On the nodes, a Linux bridge attached to a second physical NIC (a **trunk** on the switch) is created with **NMState** (`NodeNetworkConfigurationPolicy`). In the lab this bridge already existed, so the node network was not touched. Example NNCP:

```yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: br-vm
spec:
  nodeSelector:
    node-role.kubernetes.io/worker: ""
  desiredState:
    interfaces:
      - name: br-vm
        type: linux-bridge
        state: up
        bridge:
          options:
            stp:
              enabled: false
          port:
            - name: <second-physical-nic>   # must be a VLAN trunk on the switch side
              vlan:
                mode: trunk
                trunk-tags:
                  - id-range: {min: 2, max: 4094}
```

```bash
oc get nncp          # <bridge>  Available  SuccessfullyConfigured
```

> Applying an NNCP changes the node network configuration; picking the wrong port can cut the node's network connection. Try it on a single node first (with `nodeSelector`).

### 9.2 NetworkAttachmentDefinition

```bash
sed -e 's/REPLACE_ME_BRIDGE/br-vm/g' -e 's/REPLACE_ME_VLAN_ID/<vlan-id>/' nad-vm-vlan.yaml | oc apply -f -
```

**Console:** **Networking → NetworkAttachmentDefinitions → Create** → **Network Type: Linux bridge**, **Bridge name** `<bridge>`, **VLAN tag** `<vlan-id>`.

### 9.3 Adding a NIC to a running VM (hotplug)

> ⚠️ **Turn off automatic DHCP inside the VM first.** On Fedora/RHEL, NetworkManager gets an IP for a new NIC via DHCP **on its own** ("Wired connection 1"). If the corporate DHCP pool also hands out IPs that are used statically, the VM takes another server's IP the moment the NIC is attached. This happened **twice** in the lab: the first time the static second IP (a VIP) of another VM was handed out, the second time the IP of another VM that was powered off at the time.
>
> ```bash
> # inside the VM, BEFORE the NIC is attached
> printf "[main]\nno-auto-default=*\n" | sudo tee /etc/NetworkManager/conf.d/99-no-auto-default.conf
> sudo systemctl reload NetworkManager
> ```

Because the cluster's `vmRolloutStrategy` is `LiveUpdate`, the NIC is added without shutting down the VM; KubeVirt applies it with an automatic live migration in the background:

```bash
oc get kubevirt -n openshift-cnv -o jsonpath='{.items[0].spec.configuration.vmRolloutStrategy}'   # LiveUpdate
oc patch vm fedora-from-template -n ocp-poc-virt --type=json -p '[
  {"op":"add","path":"/spec/template/spec/domain/devices/interfaces/-","value":{"name":"vlan","bridge":{}}},
  {"op":"add","path":"/spec/template/spec/networks/-","value":{"name":"vlan","multus":{"networkName":"vm-vlan"}}}]'
oc get vmim -n ocp-poc-virt          # kubevirt-workload-update-xxxxx  Succeeded
```

**Console:** VM → **Configuration → Network** → **Add network interface** → **Network**: `vm-vlan`, **Type**: Bridge → **Save**.

✅ **Actual output:** the automatic migration `Succeeded`; inside the VM the new NIC (`enp2s0`) is `UP` but **has no IP** (because automatic DHCP is off).

### 9.4 Getting an IP (with conflict detection) and routing

Because the VM has two interfaces, if the VLAN interface also gets a default route, replies to requests coming from the LAN leave through the wrong interface (the NATed pod network) and the connection fails (asymmetric routing). So **no default route is taken** from the VLAN interface; only the corporate networks are routed through it. `ipv4.dad-timeout` runs an ARP conflict check before using the IP:

```bash
# inside the VM
sudo nmcli con add type ethernet ifname enp2s0 con-name vm-vlan \
  ipv4.method auto ipv4.dad-timeout 3000 ipv4.never-default yes \
  ipv4.routes "<CORP_NET>/16 <VLAN_GATEWAY>" ipv6.method disabled
sudo nmcli con up vm-vlan
ip -br -4 a show enp2s0
sudo arping -D -c 3 -w 4 -I enp2s0 <assigned-IP>      # 0 = nobody else answers
```

✅ **Actual output:**

| Check | Result |
|---|---|
| VLAN interface | `<VLAN>.181/24` from DHCP; route `<CORP_NET>/16 via <VLAN_GATEWAY> dev enp2s0` |
| `arping -D` | `rc=0` (nobody else answers) |
| Ping from another server on the corporate network | `3 received, 0% packet loss` |
| Direct SSH from that server to the VM's LAN IP | Successful |
| The VM's internet egress | Continued over the pod network (`https://quay.io` → `200`) |

In the first test round, an attempt without the route (when the VLAN interface took the default route) gave `100% loss` pinging the VM from the outside server; with the route fix it became `0% loss`.

> Even if `arping -D` finds no conflict, if the device that uses the IP statically is **powered off** at that moment, the conflict shows up later. In the second round it turned out that the IP DHCP handed out belonged to a powered-off VM, and it was released as soon as the test ended (`nmcli con down`). In a live environment, VMs should get either a **dedicated, reserved** block from the IPAM/DHCP team or static IPs.

> The IP reported by the guest agent (`oc get vmi ... .status.interfaces`) may stay stale for a few seconds after changes inside the VM. Verify from inside the VM (`ip -br a`).

---

## 10. Placement — nodeSelector, Node Affinity, VM Affinity / Anti-Affinity

VMs run inside virt-launcher pods, so all Kubernetes scheduling rules are used as-is under `spec.template.spec`. KubeVirt automatically puts the `vm.kubevirt.io/name=<vm-name>` label on every virt-launcher pod. This label can be used in VM-to-VM affinity rules.

All VMs are created with instancetype `u1.small` (1 vCPU / 2Gi) + preference `fedora`, from the `fedora` boot source. The parameters in the files are filled in for your environment:

| Parameter | Meaning | In the lab test |
|---|---|---|
| `REPLACE_ME_NODE` | Node to pin the VM to (`oc get nodes`) | One from a 3-node pool |
| `REPLACE_ME_NODE_LABEL` | Required node label (node affinity) | Role label of a second 3-node pool |
| `REPLACE_ME_PREFERRED_NODE` | Preferred node among the labeled nodes | A node from the second pool |
| `REPLACE_ME_POOL_LABEL` | Label of the **3-node** pool the anti-affinity test is limited to | Role label of the first pool |

```bash
cd scheduling
FILL='s#REPLACE_ME_NODE_LABEL#<label>#; s#REPLACE_ME_PREFERRED_NODE#<node>#; s#REPLACE_ME_POOL_LABEL#<pool-label>#; s#REPLACE_ME_NODE\b#<node>#'
for f in vm-nodeselector.yaml vm-node-affinity.yaml vm-anti-affinity.yaml; do sed -E "$FILL" $f | oc apply -f -; done
# after web-1 is Running:
oc apply -f vm-affinity.yaml

oc get vmi -n ocp-poc-virt -o custom-columns=VM:.metadata.name,PHASE:.status.phase,NODE:.status.nodeName
```

| File | Rule | Expected | ✅ Actual result |
|---|---|---|---|
| `vm-nodeselector.yaml` | `nodeSelector: kubernetes.io/hostname=<node>` | That node only | `sched-nodeselector` → **the given node** (both rounds) |
| `vm-node-affinity.yaml` | **required**: has `<label>`; **preferred** (weight 100): `<node>` | One of the labeled nodes, preferably `<node>` | `sched-node-affinity` → **the preferred node** (both rounds) |
| `vm-anti-affinity.yaml` | 4 VMs (`app=poc-web`); required node affinity: the 3-node pool; **required podAntiAffinity** (`topologyKey: kubernetes.io/hostname`) | 3 VMs on different nodes, the 4th cannot be placed | 3 VMs spread over the pool's three nodes, **one `ErrorUnschedulable`** (`web-4` in round 1, `web-3` in round 2 — which one is left out is random) |
| `vm-affinity.yaml` | **required podAffinity**: same node as `vm.kubevirt.io/name=web-1` | web-1's node | `cache-1` → **same node as web-1** (both rounds) |

✅ **Actual output (event of the VM that could not be placed):**

```
0/9 nodes are available: 3 node(s) didn't match Pod's node affinity/selector,
3 node(s) didn't match pod anti-affinity rules, 3 node(s) had untolerated taint(s).
```

(3 masters: taint; the 3 nodes outside the pool: outside node affinity; the 3 nodes in the pool: anti-affinity full.) This shows that a **required** rule is strict. If you want "spread them if possible, but run anyway if not", use `preferredDuringSchedulingIgnoredDuringExecution`.

**Toleration (reference, not applied):** the worker nodes in the lab have no taints. Because it is a shared cluster, no taint was added for testing. To place VMs on dedicated (tainted) nodes:

```yaml
spec:
  template:
    spec:
      tolerations:
        - key: dedicated
          operator: Equal
          value: virtualization
          effect: NoSchedule
```

**Console:** VM → **Configuration → Scheduling**:

- **Node selector** → add key/value.
- **Tolerations** → key/value/effect.
- **Affinity rules** → **Add affinity rule** → **Type**: Node Affinity / Workload (Pod) Affinity / Workload (Pod) Anti-Affinity. **Condition**: Required during scheduling / Preferred (weight). **Topology key**: `kubernetes.io/hostname`. Expressions are defined with label key/value.
- Changes take effect on the next placement (restart or migration).

> **Relation to live migration:** the migration target follows these rules too. A VM pinned to one node with `nodeSelector` cannot be migrated to another node (no target is found even if it is `LiveMigratable`). If anti-affinity has filled all nodes (the `web-1..3` example), migration stays `Pending` as well. Keep this in mind when planning node maintenance.

---

## 11. VM Backup and Restore With OADP

Scenario: a namespace containing a VM created from the golden template, with critical data inside, is backed up **while the VM is running**. Then the namespace is deleted completely (disaster) and restored with OADP. Data integrity is verified with SHA256.

### 11.1 DPA: the `kubevirt` and `csi` plugins

VM support is added to the `dpa-odf` created by the [16 — OADP](../../D-operations/16-oadp/README.md) guide in this repo:

```bash
oc patch dpa dpa-odf -n openshift-adp --type=merge \
  -p '{"spec":{"configuration":{"velero":{"defaultPlugins":["openshift","aws","csi","kubevirt"]}}}}'
oc rollout status deploy/velero -n openshift-adp
oc get dpa dpa-odf -n openshift-adp -o jsonpath='{range .status.conditions[*]}{.type}={.status} {.reason}{"\n"}{end}'
```

✅ **Actual output:** Velero init containers `openshift-velero-plugin`, `velero-plugin-for-aws`, `kubevirt-velero-plugin`. `--features=EnableCSI` in the Velero arguments. DPA `Reconciled=True Complete`, `VeleroReady=True`. BSL `dpa-odf-1` `Available`.

- **`kubevirt` plugin:** backs up the VM, VMI, DataVolume, PVC and virt-launcher pod as linked resources. For running VMs it freezes the file system through the guest agent before the snapshot and unfreezes it afterwards.
- **`csi` plugin:** backs up PVCs with ODF (Ceph RBD) CSI snapshots.

### 11.2 Test VM (separate namespace)

```bash
oc apply -f oadp/namespace.yaml                              # ocp-poc-virt-backup
oc create secret generic vm-ssh-key -n ocp-poc-virt-backup --from-file=key=./vmkey.pub
oc apply -f oadp/cross-namespace-clone-rbac.yaml             # the golden image is in another namespace
oc process poc-web-golden -n ocp-poc-virt -p NAME=poc-backup-vm | oc apply -n ocp-poc-virt-backup -f -
```

✅ **Actual output (without the RBAC):** the VM stayed `Stopped`. Event: `UnauthorizedDataVolumeCreate ... User system:serviceaccount:ocp-poc-virt-backup:default has insufficient permissions in clone source namespace ocp-poc-virt`. After applying `cross-namespace-clone-rbac.yaml`, the clone `Succeeded` in **9 seconds** and the VM started.

```bash
# inside the VM: critical data
echo "critical data before backup $(date -u +%FT%TZ)" | sudo tee /var/www/html/critical.txt; sync
sha256sum /var/www/html/critical.txt
```

✅ **Actual output:** `12c6bd409aca3bb6749e1a15559d82e0059b6ef09e57a40aa3a7fb8e824e95b2`, machine-id `61920a66aee74c70b57134a7fcbb65e5`.

### 11.3 Backup (while the VM is running)

`oadp/backup.yaml` specifies the CSI snapshot class **with an annotation on the Backup**:

```yaml
annotations:
  velero.io/csi-volumesnapshot-class_openshift-storage.rbd.csi.ceph.com: ocs-storagecluster-rbdplugin-snapclass
```

> Velero normally looks for a VolumeSnapshotClass labeled `velero.io/csi-volumesnapshot-class=true`. There was no such label in the lab. Instead of labeling the shared VolumeSnapshotClass, the annotation method was used; it does not touch anything cluster-wide. For permanent use, `oc label volumesnapshotclass ocs-storagecluster-rbdplugin-snapclass velero.io/csi-volumesnapshot-class=true` is more practical.

```bash
oc apply -f oadp/backup.yaml
oc get backups.velero.io poc-vm-backup-1 -n openshift-adp -w
```

> ⚠️ Because CloudNativePG is installed on this cluster, **`oc get backup` returns CNPG's `backups.postgresql.cnpg.io` resource** and gives `NotFound`. Use the full name for Velero objects: `backups.velero.io`, `restores.velero.io`.

✅ **Actual output:**

```
phase=Completed items=91/91 csi=1/1 errors= warnings= start=12:20:48Z end=12:22:21Z
hookStatus: {"hooksAttempted":2}
```

Freeze/unfreeze hooks in the Velero log: `/usr/bin/virt-freezer --freeze ...` → `Guest agent version is 10.2.2`, `Operation completed successfully`; after the snapshot `virt-freezer --unfreeze` → `Operation completed successfully`. Duration **~1.5 minutes**, the VM was not shut down.

### 11.4 Disaster simulation and restore

```bash
oc delete namespace ocp-poc-virt-backup          # deletes the VM, disk (PVC) and all resources
oc apply -f oadp/restore.yaml
oc get restores.velero.io poc-vm-restore-1 -n openshift-adp -w
```

✅ **Actual output:**

```
phase=Completed items=57/57 errors= warnings=14 start=12:34:39Z end=12:35:22Z
virtualmachine.kubevirt.io/poc-backup-vm   Running
persistentvolumeclaim/poc-backup-vm        Bound   30Gi   RWX   ocs-storagecluster-ceph-rbd-virtualization
```

The restore completed in **43 seconds** and the VM started on its own.

✅ **Second test round** (with a VM created from the golden template): backup `Completed 85/85 csi=1/1 hooksAttempted=2` in **27 s**; after the namespace was deleted, restore `Completed 50/50` in **27 s** (13 harmless warnings); the SHA256 of the data file and the machine-id were identical, httpd `active`. Without the cross-namespace clone permission the VM again waited with `UnauthorizedDataVolumeCreate`; once the RBAC was applied it started in 42 s. The disk was not cloned again; it was restored from the CSI snapshot.

**Data verification (inside the VM):**

```
critical data before backup 2026-09-29T12:20:37Z
12c6bd409aca3bb6749e1a15559d82e0059b6ef09e57a40aa3a7fb8e824e95b2  /var/www/html/critical.txt
machine-id: 61920a66aee74c70b57134a7fcbb65e5
httpd: active
```

SHA256 and machine-id are identical: the same VM came back with its data.

**14 warnings (harmless):** all of the `could not restore, ... already exists` kind. They are given for CRDs that already exist on the cluster (`virtualmachines.kubevirt.io`, `datavolumes.cdi.kubevirt.io` ...) and the SCC `kubevirt-controller`, plus the `kube-root-ca.crt`, `openshift-service-ca.crt`, `istio-ca-*` ConfigMaps, pipeline RoleBindings and dockercfg secrets that OpenShift creates automatically when a namespace is created.

**Console:** the OADP operator does not add a separate screen to the Console. Use the **Create Backup / Create Restore** forms (YAML/form) from the **Operators → Installed Operators → OADP → Backup / Restore** tabs. Status is tracked from the same tabs.

> **Scope note:** CSI snapshots are kept inside Ceph; only the Kubernetes objects are written to S3. So this method is enough for restoring **into the same cluster**. If Ceph itself is lost or the VM will move to another cluster, turn on `nodeAgent` in the DPA and use `snapshotMoveData: true` (Data Mover) in the Backup. The disk data is then also copied to S3 (ODF RGW).

---

## 12. Known Limitations / Findings From the Live Test

- **IP conflict with DHCP (important):** in the first attempt on the corporate VLAN, DHCP gave the test VM an address that was defined on another VM as a **static second IP (VIP)** (`arping -D` got a reply from another MAC; SSH from outside landed on another server). In the second round the IP handed out also belonged to another VM that was powered off at the time. In both cases the IP was released right away (see 9.3–9.4). **Lesson:** static IPs must be reserved / excluded on the DHCP server. When giving VMs LAN IPs in a live environment, either get a dedicated block from the IPAM/DHCP team or check every IP with `arping -D` before use.
- **Automatic DHCP on Multus NICs:** in Fedora/RHEL images, NetworkManager gets a DHCP IP for a newly added NIC on its own ("Wired connection 1"). Turn it off with `no-auto-default=*` before the hotplug (see 9.3).
- **Hotplug disk + SELinux snapshot error:** if the mount point of a freshly formatted disk stays unlabeled, the guest agent cannot freeze the file system and the snapshot becomes `Failed`; fixed with `restorecon -v <mount-point>` (see 8).
- **`oc get subscription` ambiguity:** if ACM is installed, the `subscription` short name returns ACM's `subscriptions.apps.open-cluster-management.io` resource. For OLM use `oc get subscriptions.operators.coreos.com`.
- **Hotplug disks are SCSI:** they appear as `/dev/sdX`, not `virtio` (see 6.1).
- **`runStrategy: Always` + in-guest `poweroff`:** the VM is restarted. For a permanent shutdown use `virtctl stop` / Console **Stop**.
- **`oc wait vmi` after `virtctl start`:** the VMI object is created a few seconds later. Running `oc wait vmi` immediately returns `NotFound`. Use `oc wait vm <name> --for=condition=Ready` or add a short wait.
- **Windows ISO install:** the Windows installer does not recognize virtio disk/network drivers. The `virtio-win` container disk provided by OpenShift Virtualization must be attached as a second CD-ROM (the "Mount Windows drivers disk" checkbox in the Console). For live-tested details see [17.1 — Windows Boot Source](windows/README.md).
- **Golden image generalization:** a cloned disk also carries identities such as hostname, SSH host keys and machine-id (see section 5).
- **Cross-namespace disk clones need RBAC:** if the golden image/template is in another namespace, you get `UnauthorizedDataVolumeCreate`. Grant `datavolumes/source` permission in the source namespace to the target namespace's ServiceAccount (`oadp/cross-namespace-clone-rbac.yaml`).
- **`oc get backup` ambiguity:** if operators with a CRD named `Backup` (such as CloudNativePG) are installed, `oc get backup` does not return Velero. Use `backups.velero.io` / `restores.velero.io`.

---

## 13. Cleanup

```bash
oc delete -f scheduling/ --ignore-not-found
oc delete vm alpine-from-golden alpine-from-iso fedora-from-template poc-vm-from-custom-template -n ocp-poc-virt
oc delete vmrestore,vmsnapshot --all -n ocp-poc-virt
oc delete template poc-fedora-small poc-web-golden -n ocp-poc-virt
oc delete namespace ocp-poc-virt ocp-poc-virt-backup
oc delete clusterrole poc-datavolume-cloner
oc delete restores.velero.io poc-vm-restore-1 -n openshift-adp
# To delete the backup together with its S3 data and CSI snapshots (otherwise it is deleted by its TTL after 72 hours):
oc create -f - <<'EOF'
apiVersion: velero.io/v1
kind: DeleteBackupRequest
metadata:
  generateName: poc-vm-backup-1-delete-
  namespace: openshift-adp
spec:
  backupName: poc-vm-backup-1
EOF
```

To revert the plugin addition in `dpa-odf` (back to its original state from the [16 — OADP](../../D-operations/16-oadp/README.md) guide):

```bash
oc patch dpa dpa-odf -n openshift-adp --type=merge \
  -p '{"spec":{"configuration":{"velero":{"defaultPlugins":["openshift","aws"]}}}}'
```

---

## 14. Windows VMs

Windows templates come with the installation, but because of Microsoft licensing **no Windows boot source (image) is shipped**; the organization must provide it. Four methods (uploading a ready image, installing from ISO, a Tekton pipeline, registry + `DataImportCron`) were tested live with Windows Server 2022 and are collected in a separate guide: **[17.1 — Windows Boot Source](windows/README.md)**.

---

## Summary

| Scenario | CLI | Console | Live test |
|---|---|---|---|
| Template listing / export / parameters | `oc get template`, `oc get -o yaml`, `oc process --parameters` | Virtualization → Templates | ✅ |
| VM from a template | `oc process ... \| oc apply` | Catalog → Template catalog | ✅ Running in 20 s |
| Custom template | `custom-template.yaml` | Templates → Clone | ✅ With an SSH key parameter |
| Template from an existing VM | `vm-to-template.sh` | Bootable volumes + Templates → Clone | ✅ 2 VMs started; customizations carried over, identities regenerated |
| ISO upload | `virtctl image-upload` | Bootable volumes → Upload | ✅ 66 MB / 37 s |
| Install from ISO | `vm-from-iso.yaml` + console | Boot from CD | ✅ Install + CD-ROM eject + boot from disk |
| Golden image | `golden-image.yaml` | Bootable volumes | ✅ |
| Hotplug disk / online expansion | `virtctl addvolume`, `oc patch pvc` | Storage → Add disk / Expand PVC | ✅ 10→20Gi, data kept |
| Live migration | `virtctl migrate` / `VirtualMachineInstanceMigration` | Actions → Migrate | ✅ 7 s, 0 packet loss |
| Snapshot / restore | `VirtualMachineSnapshot` / `VirtualMachineRestore` | Snapshots tab | ✅ Online + guest agent freeze |
| Multus VLAN NIC | `nad-vm-vlan.yaml` + patch (hotplug) | Network → Add interface | ✅ Direct SSH to the LAN IP |
| nodeSelector / node affinity | `scheduling/*.yaml` | Scheduling tab | ✅ |
| VM affinity / anti-affinity | `scheduling/*.yaml` | Scheduling → Affinity rules | ✅ 4th VM Unschedulable |
| VM backup / restore with OADP | `oadp/backup.yaml`, `oadp/restore.yaml` | Installed Operators → OADP | ✅ Backup while running (freeze), namespace deleted, restore 43 s, SHA256 identical |
| Windows boot source (4 methods) | `windows/` | Bootable volumes, Template catalog, Pipelines | ✅ With all four methods a VM started from the ready-made `windows2k22-server-medium` template, reachable over RDP |
