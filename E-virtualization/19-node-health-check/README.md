# 19 — Automatic Recovery on Node Loss (NodeHealthCheck + Self Node Remediation)

> 🇹🇷 [Türkçe](README.tr.md) · [← 18 — Migration with MTV](../18-mtv/README.md) · [POC flow](../../README.md)

When a worker node suddenly dies (power loss, kernel panic, network cut), Kubernetes **does not move the pods and VMs on that node elsewhere by itself**. It cannot be sure the node is really down; if the same VM ran on two nodes at once, the disk could be corrupted. This guide sets up **Node Health Check (NHC)** + **Self Node Remediation (SNR)**, which automatically **fences** an unhealthy node (isolates and reboots it) so workloads can safely start on another node.

Every step was **tested end to end on a lab cluster** (OpenShift 4.22, OpenShift Virtualization 4.22.9, NHC 0.12.1, SNR 0.13.1, bare-metal workers): the kubelet on a worker node was stopped on purpose, NHC/SNR rebooted the node, the VM on it started on another node, and the node rejoined the cluster on its own.

Scenario order:

1. Concepts: why automatic fencing is needed
2. Installing the operators
3. The NodeHealthCheck rule
4. Test VM
5. Taking the node down and the timeline
6. Tuning recommendations and things to watch
7. Cleanup

---

## 1. Concepts

| Component | Role |
|---|---|
| **NodeHealthCheck (NHC)** | Watches the selected nodes; if a node stays `Ready=False/Unknown` for the configured time, it starts remediation |
| **Self Node Remediation (SNR)** | An agent running on every node. The node remediation is opened for **reboots itself** (via watchdog / software reboot). The other nodes add the `out-of-service` taint to it once the safe wait time has passed |
| `out-of-service` taint | Tells Kubernetes "this node is definitely down; you can clean up its pods and volume attachments"; StatefulSet pods and VMs start on another node only after this |
| **Fence Agents Remediation (FAR)** | Alternative to SNR: powers the node off and on through IPMI/iLO/iDRAC (BMC). Faster and more certain if BMC access is available |

**Why is it needed?** Without NHC/SNR, when a node's kubelet or network goes away, the VM stays `Running@<dead-node>` and **does not start elsewhere**; to prevent two copies from using the same disk at once (split-brain), KubeVirt must be sure the node has been fenced.

---

## 2. Installing the Operators

They are in the Red Hat catalog (`redhat-operators`). Packages with the same names may also exist in the community catalog; make sure `source: redhat-operators`:

```bash
oc get packagemanifests -n openshift-marketplace -l catalog=redhat-operators | grep -E 'node-healthcheck|self-node-remediation'
oc apply -f operators.yaml          # namespace openshift-workload-availability + 2 Subscriptions
oc get csv -n openshift-workload-availability
oc get selfnoderemediationtemplate -n openshift-workload-availability
oc get ds self-node-remediation-ds -n openshift-workload-availability
```

✅ **Actual output:** `node-healthcheck-operator.v0.12.1 Succeeded`, `self-node-remediation.v0.13.1 Succeeded` (~1 min). The `self-node-remediation-automatic-strategy-template` template and the agent DaemonSet (`9/9` nodes) were created automatically; `SelfNodeRemediationConfig` uses `/dev/watchdog` as the watchdog.

**Console:** **Operators → OperatorHub** → "Node Health Check Operator" (SNR is installed automatically as a dependency) → **Install**. Then **Compute → NodeHealthChecks → Create**.

---

## 3. The NodeHealthCheck Rule

`nodehealthcheck.yaml`:

- **Selector:** workers only; the `master`/`control-plane` roles are **explicitly excluded**. Rebooting control plane nodes automatically puts etcd quorum at risk.
- **`minHealthy: 51%`:** if more than half of the selected nodes are unhealthy at the same time (e.g. a switch failure), NHC **does not intervene**; this prevents a mass reboot from making things worse.
- **`unhealthyConditions`:** remediation starts if `Ready` stays `False` or `Unknown` for 300 s.

```bash
oc apply -f nodehealthcheck.yaml
oc get nodehealthcheck poc-worker-nhc -o jsonpath='{.status.phase} observed={.status.observedNodes} healthy={.status.healthyNodes}'
```

> **To limit the blast radius during a test**, a single node pool's label can be added to the selector (the commented line in the file). In the lab test the rule was limited to a 3-node pool.

✅ **Actual output:** `phase=Enabled observed=3 healthy=3 reason=NHC is enabled, no ongoing remediation`.

---

## 4. Test VM

`test-vm.yaml`: `runStrategy: Always` (the VM is restarted if it stops) and an **RWX** disk (`ocs-storagecluster-ceph-rbd-virtualization`). RWX lets the disk attach to another node quickly.

```bash
oc create namespace poc-ha-demo
oc apply -f test-vm.yaml
oc get vmi ha-test-vm -n poc-ha-demo -o jsonpath='{.status.nodeName}'
```

For the test, a `preferredDuringScheduling` node affinity was added so the VM would land on the node to be taken down (don't use `required`; otherwise the VM cannot move to another node).

✅ **Actual output:** `VM: Running node=<worker-A>`.

---

## 5. Taking the Node Down and the Timeline

A "node unreachable" situation was simulated by stopping the node's kubelet (the machine is on, but its communication with the cluster is cut — the hardest case):

```bash
oc debug node/<worker-A> --to-namespace=default -- chroot /host \
  systemd-run --on-active=15 --unit=ha-test systemctl stop kubelet.service
```

> For a real power-loss test, the node can be powered off through the BMC/hypervisor; the flow is the same.

Monitoring (every 10 s): node `Ready` status, taints, NHC phase, SNR objects and where the VM is.

✅ **Actual output (from the moment the kubelet was stopped, at 10 s resolution):**

| Time | Event |
|---|---|
| 0 s | kubelet stopped |
| ~65 s | Node `Ready=Unknown`, `node.kubernetes.io/unreachable` taint. VM still `Running@worker-A` (correct: it is not moved before the node is fenced) |
| ~370 s | `Unknown` passed 300 s → NHC `Remediating`, `SelfNodeRemediation` created, `remediation.medik8s.io/self-node-remediation` taint |
| ~375 s | The SNR agent **rebooted** the node (the last log line of the node's previous boot was seconds after remediation started) |
| ~490 s | SNR safe wait time passed → `node.kubernetes.io/out-of-service` taint |
| ~510 s | VM rescheduled (`Scheduling`) |
| **~525 s** | **VM `Running@worker-B`**, then the guest agent connected (`Fedora Linux 44`) |
| ~590 s | worker-A rebooted and became `Ready`; taints removed, SNR object deleted, NHC back to `Enabled` |

Total time from node loss to the VM running on another node: **~8 min 45 s**; 5 minutes of that is NHC's `duration: 300s` wait (section 6).

---

## 6. Tuning Recommendations and Things to Watch

- **Shortening the time:** most of the total time is `unhealthyConditions[].duration` (300 s) and SNR's safe wait time. `duration` can be lowered to 60–120 s, but the risk of unnecessary reboots on short network blips goes up. If BMC access is available, **Fence Agents Remediation** powers the node off directly, so the wait is shorter.
- **Never include the control plane:** `master`/`control-plane` must be excluded in the selector. A master rebooting on its own can leave etcd in the minority.
- **`minHealthy`:** in a network or switch failure all nodes go `Unknown` at once; thanks to `minHealthy`, NHC does not do a mass reboot.
- **VMs must use `runStrategy: Always`** (or `RerunOnFailure`); `Manual` VMs don't start on another node by themselves.
- **RWX disk:** the RWX disk needed for live migration also makes it easier to attach the disk to the new node during recovery.
- **Without NHC:** in the same scenario the VM stays `Running@<dead-node>` until the node is powered off by hand and the `out-of-service` taint is added by hand.
- **Leftovers:** when the operator is removed, the CRDs (`*.medik8s.io`), the `node-remediation-console-plugin` ConsolePlugin and some ClusterRoles stay behind (section 7).

---

## 7. Cleanup

```bash
oc delete project poc-ha-demo
oc delete nodehealthcheck poc-worker-nhc
for s in node-healthcheck-operator self-node-remediation; do
  csv=$(oc get subscriptions.operators.coreos.com $s -n openshift-workload-availability -o jsonpath='{.status.installedCSV}')
  oc delete subscriptions.operators.coreos.com $s -n openshift-workload-availability
  oc delete csv $csv -n openshift-workload-availability
done
oc delete namespace openshift-workload-availability
# Leftovers OLM does not delete:
oc get crd -o name | grep medik8s | xargs -r oc delete
oc delete consoleplugin node-remediation-console-plugin
oc delete clusterrole node-healthcheck-metrics-reader node-healthcheck-operator-aggregation \
  self-node-remediation-ext-remediation self-node-remediation-metrics-reader
oc delete clusterrolebinding node-healthcheck-operator-aggregation
```

✅ **Actual output:** all of the leftovers above were still on the cluster after the operator was removed and were deleted by hand.

---

## Summary

| Step | Result |
|---|---|
| NHC + SNR installation | ✅ ~1 min, agent on all nodes |
| NHC rule (workers, masters excluded, `minHealthy 51%`) | ✅ `Enabled` |
| Node unreachable (kubelet stopped) | ✅ `Unknown` in ~65 s |
| NHC remediation + SNR reboot | ✅ after 300 s |
| VM `Running` on another node | ✅ **~8 min 45 s** (total) |
| Node rejoining on its own | ✅ ~10 min (590 s), taints removed automatically |
