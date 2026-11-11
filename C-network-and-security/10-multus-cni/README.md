# 10 — Multus CNI — Macvlan Demo Guide

> 🇹🇷 [Türkçe](README.tr.md) · [← 09 — Micro-Segmentation](../09-micro-segmentation/README.md) · [POC flow](../../README.md) · [11 — Security Tests →](../11-security-tests/README.md)

This document shows how to use Multus CNI + macvlan to add a **second network interface** to a pod (one that gets an IP from the real LAN, the same segment as the node's physical network). Every step was **tested end to end on a lab cluster (OpenShift 4.22, **OVNKubernetes**, bare-metal workers) in the `ocp-poc` namespace, on the real LAN where the nodes live**. The IP block used was scanned with `arping` and confirmed free before going live. In the output, lab network addresses are shortened like `<LAN>.241` (`<LAN>` = the nodes' /24 subnet, `<REMOTE>` = another subnet behind a router).

Scenario order:

1. What Multus/macvlan is and what it's for
2. Prerequisite check (is Multus already active, which node interface to use)
3. Creating a `NetworkAttachmentDefinition` (macvlan + whereabouts IPAM)
4. Running a pod with the second interface
5. Verification (same node, different node)
6. Access from a different subnet (behind a router) — `rp_filter` and the fix
7. Known limitations / common mistakes
8. Cleanup

---

## 1. What Are Multus / Macvlan?

- **Multus** is a "meta-CNI" plugin that lets you attach more than one network interface (NIC) to a pod. On OpenShift it is installed and active by default — it lets you define **extra** interface(s) for the pods that need them, without breaking Kubernetes' standard single-interface (`eth0`, cluster network) model.
- **Macvlan** is a CNI plugin used for one of these extra interfaces: it gives the pod's extra interface its own MAC + IP address from the same L2 segment (the real LAN) as the node's **physical** NIC. Result: independently of the cluster's SDN (OVN), the pod can talk directly to other machines on the LAN as if it were a device plugged straight into that physical switch.
- Typical uses: legacy/on-prem applications that need a real LAN IP, VM/NFV-like workloads, protocols that need multicast/broadcast, and cases where the real source IP must be visible where the SDN would NAT it.

> **Important:** macvlan is added **next to**, not **instead of**, the cluster's normal `eth0` (OVN) interface. The pod still carries its in-cluster traffic over `eth0` with normal Services/Routes/NetworkPolicies; macvlan only opens an extra path.

---

## 2. Prerequisite Check

**a) Is Multus / multi-network already active?**

```bash
oc get network.operator cluster -o jsonpath='{.spec.disableMultiNetwork}{"\n"}'
# Expected: false (or if the field is missing, the default is already false = active)
```

On OpenShift this is **on by default** — no separate operator install is needed.

**b) Which node interface will be the "master"?**

This step is critical and contains an **OVNKubernetes-specific pitfall** (detailed in section 7). Find the physical NIC on the node:

```bash
oc debug node/<node-name> -- chroot /host ip -br link show
```

On nodes managed by OVNKubernetes, the primary physical NIC (e.g. `eno1`) is **enslaved to the OVS `br-ex` bridge** — so giving `eno1` directly as the macvlan master often does not work as expected. Use **`br-ex`** as the master instead:

```bash
oc debug node/<node-name> -- chroot /host ip -br addr show br-ex
# confirm that br-ex carries the node's real LAN IP
```

**c) Is whereabouts installed for IPAM?**

```bash
oc get crd ippools.whereabouts.cni.cncf.io
```

This CRD ships with OpenShift's Multus package; no separate install is needed. `whereabouts` is an IPAM plugin that prevents static IP collisions cluster-wide (even when pods start on several nodes at the same time) — **strongly recommended** for "IP from the real LAN" scenarios like macvlan (otherwise two pods can get the same IP and cause an IP conflict on the LAN).

---

## 3. Creating the NetworkAttachmentDefinition

A `NetworkAttachmentDefinition` (NAD) is a namespaced CRD that defines the extra network your pods will reference.

```yaml
# macvlan-nad.yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: macvlan-lan
  namespace: REPLACE_ME_NAMESPACE
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "type": "macvlan",
      "master": "br-ex",
      "mode": "bridge",
      "ipam": {
        "type": "whereabouts",
        "range": "REPLACE_ME_CIDR",
        "range_start": "REPLACE_ME_IP_START",
        "range_end": "REPLACE_ME_IP_END",
        "gateway": "REPLACE_ME_GATEWAY"
      }
    }
```

What to fill in (confirm with the network team — it must be a block that is **unused on the real LAN and outside the DHCP pool**, otherwise you get IP conflicts):

| Placeholder | Meaning | In the lab test |
|---|---|---|
| `REPLACE_ME_NAMESPACE` | The pods' namespace | `ocp-poc` |
| `REPLACE_ME_CIDR` | The LAN subnet where the nodes (br-ex) live | `<LAN>.0/24` |
| `REPLACE_ME_IP_START` / `REPLACE_ME_IP_END` | A free block on the LAN to give to pods | `<LAN>.241` – `<LAN>.245` |
| `REPLACE_ME_GATEWAY` | The LAN's default gateway | `<LAN>.1` |

```bash
# Easy fill-in example:
sed -e 's/REPLACE_ME_NAMESPACE/ocp-poc/' -e 's#REPLACE_ME_CIDR#192.168.10.0/24#' \
    -e 's/REPLACE_ME_IP_START/192.168.10.241/' -e 's/REPLACE_ME_IP_END/192.168.10.245/' \
    -e 's/REPLACE_ME_GATEWAY/192.168.10.1/' macvlan-nad.yaml | oc apply -f -
```

**Confirm the IP block is free before going live** (ICMP alone is not enough, a device may be powered off — the ARP table + `arping` at L2 is more reliable):

```bash
# Check the current ARP table from the node
oc debug node/<node-name> --to-namespace=default -- chroot /host ip neigh show dev br-ex

# Probe the candidate range at L2 with arping (no reply = considered free)
oc debug node/<node-name> --to-namespace=default -- chroot /host bash -c '
for i in $(seq 241 245); do
  echo "<LAN>.$i -> $(arping -c2 -w2 -I br-ex <LAN>.$i | grep -ci "reply from") replies"
done'
```

✅ **Actual output:** `0 replies` for all five candidates → the block is free.

> `--to-namespace=default`: by default `oc debug node` starts a temporary pod in the current project namespace. If that namespace has been deleted, it fails with `unable to get namespace`; passing the namespace explicitly avoids that.

```bash
oc apply -f macvlan-nad.yaml
```

> **`mode: bridge`** was chosen because it lets several macvlan pods attached to the same physical interface **also talk to each other**. Other modes such as `mode: private`/`vepa` restrict pod-to-pod traffic or make it depend on the switch — for a LAN demo, `bridge` is the least troublesome option.

---

## 4. Running a Pod With the Second Interface

The NAD is referenced in the pod's `metadata.annotations`:

```yaml
# macvlan-test-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: macvlan-test
  namespace: REPLACE_ME_NAMESPACE
  annotations:
    k8s.v1.cni.cncf.io/networks: macvlan-lan
spec:
  containers:
    - name: test
      image: busybox:1.36
      command: ["sleep", "3600"]
```

```bash
oc apply -f macvlan-test-pod.yaml
oc wait --for=condition=Ready pod/macvlan-test -n REPLACE_ME_NAMESPACE --timeout=60s
```

---

## 5. Verification (actual test output)

In the test, the two pods were placed on **different nodes** (`spec.nodeName`), so pod-to-pod traffic went over the physical LAN/switch.

**a) The extra interface and IP assigned to the pod:**

```bash
oc exec -n ocp-poc macvlan-test -- ip addr show net1
```

✅ **Actual output:**

```
3: net1@if14: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue qlen 1000
    link/ether ce:0a:ab:f9:29:0f brd ff:ff:ff:ff:ff:ff
    inet <LAN>.241/24 brd <LAN>.255 scope global net1
```

| Pod | Node | `eth0` (OVN) | `net1` (macvlan/LAN) |
|---|---|---|---|
| `macvlan-test` | worker-A | `10.129.x.x` | `<LAN>.241/24` |
| `macvlan-test-2` | worker-B | `10.131.x.x` | `<LAN>.242/24` |

`eth0` keeps carrying the cluster (OVN) IP; `net1` carries the real LAN IP from whereabouts — both interfaces coexist, neither replaces the other. The pod's **default route stays on `eth0`**; `net1` only adds a route for its own subnet.

**b) Two pods attached to the same NAD (on different nodes) reaching each other over macvlan:**

```bash
# Start the listener in the background (if output is not redirected, the oc exec session never closes and the command hangs)
oc exec -n ocp-poc macvlan-test-2 -- sh -c "nc -l -p 5000 > /tmp/out.txt 2>/dev/null </dev/null &"
oc exec -n ocp-poc macvlan-test   -- sh -c "echo hello | nc -w3 <macvlan-test-2 net1 IP> 5000"
oc exec -n ocp-poc macvlan-test-2 -- cat /tmp/out.txt
```

✅ **Actual output:** `hello-cross-node` — the two pods on different nodes (`<LAN>.241` → `<LAN>.242`) communicated over TCP across the physical LAN. (In an earlier test round, two pods on the same node also talked without issues.)

> **Note:** busybox's `ping` returns `permission denied (are you root?)` under this SCC (ICMP raw sockets are disabled in the restricted SCC) — do connectivity tests with `nc` (TCP).

---

## 6. Access From a Different Subnet (Behind a Router) — `rp_filter` and the Fix

Devices on the same LAN reach the pod's `net1` IP without trouble. But requests from **another subnet behind a router** go **unanswered** by default.

✅ **Actual output (the problem):** from a server in the `<REMOTE>` subnet, `nc <LAN>.241 6000` → `Ncat: TIMEOUT`. In the pod, `cat /proc/sys/net/ipv4/conf/net1/rp_filter` → `1`.

**Root cause:** the pod's default route is on `eth0` (OVN). The reply to a packet arriving at `net1` from `<REMOTE>` would have to leave through `eth0`; `rp_filter=1` (strict reverse-path filtering) sees this asymmetry and **silently drops** the packet.

### Fix (recommended): add a route to the IPAM — no node change needed

Give the pod a route "to the `<REMOTE>` subnet via `net1`'s gateway". The incoming packet and the reply then use the same interface, and `rp_filter` accepts the packet. `macvlan-nad-routed.yaml`:

```json
"ipam": {
  "type": "whereabouts",
  "range": "REPLACE_ME_CIDR",
  "range_start": "REPLACE_ME_IP_START",
  "range_end": "REPLACE_ME_IP_END",
  "gateway": "REPLACE_ME_GATEWAY",
  "routes": [
    { "dst": "REPLACE_ME_REMOTE_CIDR", "gw": "REPLACE_ME_GATEWAY" }
  ]
}
```

`REPLACE_ME_REMOTE_CIDR`: the subnet(s) where the clients that will reach the pod live. For several subnets add more lines to the list; for an organization's whole internal network, e.g. `10.0.0.0/8`.

```bash
oc apply -f <filled-in macvlan-nad-routed.yaml>
# Pod annotation: k8s.v1.cni.cncf.io/networks: macvlan-lan-routed  (the pod must be recreated)
oc exec -n ocp-poc macvlan-test -- ip route
```

✅ **Actual output (the fix):**

```
default via 10.129.2.1 dev eth0
<REMOTE>.0/24 via <LAN>.1 dev net1          <-- route from the IPAM
<LAN>.0/24 dev net1 scope link src <LAN>.241
```

From the server in the `<REMOTE>` subnet: `nc <LAN>.241 6000` → **`pod1-reply`**, `ping <LAN>.241` → **`3 packets transmitted, 3 received, 0% packet loss`**. The nodes were not touched and no reboot was needed.

### Alternatives (not recommended)

- **Relaxing `rp_filter` with the `tuning` CNI plugin:** OpenShift blocks this at admission; the pod stays in `ContainerCreating`:
  ```
  plugin type="tuning" failed (add): Sysctl net.ipv4.conf.IFNAME.rp_filter is not allowed.
  Only the following sysctls are allowed: [^net.ipv4.conf.IFNAME.accept_redirects$ ...]
  ```
- **Node-wide `net.ipv4.conf.default.rp_filter=2` with a MachineConfig:** works, but the MCO **reboots** the affected nodes one by one and impacts every pod on them. It needs the platform team's approval; not needed when the route fix above exists.

---

## 7. Known Limitations / Common Mistakes (confirmed in the live test)

- **`br-ex`, not `eno1` — an OVNKubernetes-specific pitfall:** on this cluster the primary physical NIC is enslaved to the OVS `br-ex` bridge. If you give the physical NIC name (`eno1`) directly as the master, the NAD **is created, but** the pod's macvlan interface either sees no traffic at all or behaves unexpectedly. **Always use `br-ex` as the master.**

- **Macvlan's own host–pod isolation limit:** the node itself (host netns) **cannot ping** a macvlan pod running on it — confirmed in the live test too (ping from the pod's node to the pod's `net1` IP gave `100% loss`). This is normal, expected behavior by design of the Linux macvlan driver (the parent interface cannot reach its own children directly), not a bug.
- **IP conflict risk:** if a fixed/manual IP (`"ipam": {"type": "static", ...}`) is used instead of `whereabouts`, parallel pods on several nodes can get the same IP (each node keeps its own CNI IPAM state; there is no cluster-wide coordination) — causing IP conflicts on the LAN. So for macvlan + real LAN, **treat whereabouts as mandatory**.
- **If the nodes are VMs (VMware, etc.) — promiscuous mode / MAC settings:** each macvlan interface talks with its own MAC address. If the nodes run as VMs on a hypervisor, **"MAC Address Changes"** and **"Forged Transmits"** (and "Promiscuous Mode" if needed) must be allowed at the port group / vSwitch level; otherwise the hypervisor drops the pods' traffic. This is configured outside OpenShift, in the virtualization layer. Inside the node, `ip -d link show br-ex` → the `promiscuity` value shows the Linux/OVS side is ready. Bare-metal nodes don't have this problem.
- **NetworkPolicy does not cover macvlan:** Kubernetes `NetworkPolicy` resources only cover the cluster's primary (OVN) interface; they **do not restrict** traffic in or out over macvlan (`net1`). Access control for macvlan pods must be handled at the LAN/switch/firewall level.

---

## 8. Cleanup

```bash
oc delete pod macvlan-test macvlan-test-2 -n ocp-poc
oc delete net-attach-def macvlan-lan macvlan-lan-routed -n ocp-poc
```

---

## Summary

| Component | Role |
|---|---|
| `NetworkAttachmentDefinition` | Defines the extra network (which CNI plugin, which master interface, which IPAM) |
| `k8s.v1.cni.cncf.io/networks` annotation | Attaches the pod to a specific NAD |
| `master: br-ex` | The right target on OVNKubernetes nodes for macvlan to reach the real physical NIC |
| `mode: bridge` | Lets pods attached to the same NAD also talk to each other |
| `whereabouts` IPAM | Coordinated cluster-wide IP allocation — prevents conflicts |
| `net1` | The name of the macvlan interface inside the pod (`eth0` = cluster/OVN, `net1` = macvlan/LAN) |

**Golden rules:**
1. On OVNKubernetes clusters the macvlan master must be **`br-ex`, not the physical NIC**.
2. For macvlan demos that reach the real LAN, **always** use whereabouts as IPAM; don't hand out static IPs manually.
3. Not being able to ping the node itself from the pod over macvlan is not a bug; it is a natural result of the kernel's macvlan design.
4. NetworkPolicy does not cover macvlan traffic — plan access control at the LAN level.
5. **Inbound access to the pod from a different subnet does not work by default because of `rp_filter=1`.** The fix is to add a **route** for those subnet(s) via the `net1` gateway in the NAD's IPAM (`macvlan-nad-routed.yaml`); no node change or reboot needed.
6. The pod IP block must be **outside the DHCP pool** and checked with `arping` before use.
