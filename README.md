# OpenShift POC Kit

> 🇹🇷 [Türkçe](README.tr.md)

**19 hands-on OpenShift proof-of-concept guides, every one of them run end to end on a real cluster.** Each guide gives the YAML, the `oc` / `virtctl` commands, the Web Console steps, and the **actual output** that came back — including the parts that broke and how they were fixed.

Most OpenShift tutorials stop at "apply this and it works". These guides keep going: what the router delay really is during a Blue/Green switch, why your egress NetworkPolicy silently breaks DNS, why `oc get backup` returns the wrong resource, why a Windows image from a registry boots to a blank screen. Those findings are marked **"✅ Actual output"** in every guide.

Tested on: OpenShift 4.22 · OpenShift Virtualization 4.22 · ODF 4.22 · OpenShift GitOps 1.22 · OpenShift Pipelines 1.23 · Logging 6.6 · OADP 1.6 · RHACS 4.11 · ACM · MTV 2.12 · bare-metal workers, OVNKubernetes.

## Flow

The folders are grouped **in the order you should run them** (A–E) and numbered (01–19). The order follows the dependencies between guides: platform foundation → application delivery → network and security → operations → virtualization. A resource created by one guide (a namespace, the DPA, Loki, etc.) is used by later ones, so going in order is recommended.

| # | Guide | Topic | Prerequisite / dependency |
|---|---|---|---|
| **A** | **[Platform Foundation](A-platform-foundation)** | | |
| 01 | [Identity and Access](A-platform-foundation/01-identity-and-access/README.md) | LDAP/AD and HTPasswd identity providers, group sync, RBAC (dev/prod, role-based access) | cluster-admin |
| 02 | [Container Management](A-platform-foundation/02-container-management/README.md) | Namespace/Project, rollout/rollback, HPA, ResourceQuota, LimitRange | — (**this guide creates** the `ocp-poc` namespace; 03, 04, 10 and 12 use it) |
| 03 | [Application Scenarios](A-platform-foundation/03-application-scenarios/README.md) | Stateless, stateful (ODF), API/microservice | 02 (02's quota must be removed) |
| 04 | [Image Management](A-platform-foundation/04-image-management/README.md) | Internal/external registry, private pull secret, vulnerability scanning with ACS | 02, RHACS |
| **B** | **[Application Delivery](B-application-delivery)** | | |
| 05 | [CI/CD](B-application-delivery/05-ci-cd/README.md) | OpenShift Pipelines + GitOps (Argo CD) | OpenShift Pipelines, OpenShift GitOps, access to this repo |
| 06 | [Blue/Green Deployment](B-application-delivery/06-blue-green-deployment/README.md) | Two Deployments + one Route, instant switch | 05 (conceptual follow-up) |
| 07 | [Canary Deployment](B-application-delivery/07-canary-deployment/README.md) | Gradual shift with a weighted Route | 06 |
| **C** | **[Network and Security](C-network-and-security)** | | |
| 08 | [NetworkPolicy](C-network-and-security/08-network-policy/README.md) | Default deny, namespace/pod-based allow, egress | — |
| 09 | [Micro-Segmentation](C-network-and-security/09-micro-segmentation/README.md) | Tier/role-based isolation | 08 |
| 10 | [Multus CNI](C-network-and-security/10-multus-cni/README.md) | A second network interface for a pod (macvlan) | 02 |
| 11 | [Security Tests](C-network-and-security/11-security-tests/README.md) | SCC/PSA, image signing | ImagePolicy rolls out an MCP to all nodes (~1.5 min, no reboot); needs cosign v2 and `MatchRepository` |
| **D** | **[Operations](D-operations)** | | |
| 12 | [Logging + LokiStack](D-operations/12-logging/README.md) | Application/audit logs → Loki (ODF S3) | ODF (Ceph RGW), 02 |
| 13 | [East-West Traffic Control](D-operations/13-east-west-traffic-control/README.md) | NetworkPolicy + OVN ACL logging | 08, 09, 12 |
| 14 | [MultiCluster Observability](D-operations/14-multicluster-observability/README.md) | ACM Observability (Thanos, ODF S3) | ACM (MultiClusterHub), ODF |
| 15 | [ODF Performance Test](D-operations/15-odf-performance-test/README.md) | IOPS / throughput / latency with fio | ODF |
| 16 | [OADP Backup/Restore](D-operations/16-oadp/README.md) | Namespace backup/restore with Velero (ODF S3) | ODF (Ceph RGW). **This guide creates** `dpa-odf`; 17 uses it |
| **E** | **[Virtualization](E-virtualization)** | | |
| 17 | [OpenShift Virtualization](E-virtualization/17-virtualization/README.md) | Templates, ISO, golden image, live migration, snapshot, Multus VLAN, affinity, VM backup | OpenShift Virtualization, ODF, NMState, 16 |
| 17.1 | ↳ [Windows Boot Source](E-virtualization/17-virtualization/windows/README.md) | 4 ways to provide images for Windows templates | 17, OpenShift Pipelines (for method 3) |
| 18 | [Migration with MTV](E-virtualization/18-mtv/README.md) | Migrating VMs from VMware vSphere / OVA (provider, maps, plan) | Migration Toolkit for Virtualization, 17 |
| 19 | [Automatic Recovery on Node Loss](E-virtualization/19-node-health-check/README.md) | NodeHealthCheck + Self Node Remediation: fence a dead node and restart its VMs elsewhere | 17; worker nodes (control plane excluded) |

## Before the POC

At the start of the day, confirm that the cluster and the operators you will use are ready:

```bash
oc get clusterversion
oc get nodes
oc get co | awk 'NR==1 || $3!="True" || $4!="False" || $5!="False"'   # problematic ClusterOperators
oc get storagecluster -n openshift-storage                               # ODF: Ready
oc get csv -A --no-headers -o custom-columns=CSV:.metadata.name,PHASE:.status.phase \
  | grep -iE '^(openshift-pipelines|openshift-gitops|rhacs|loki|cluster-logging|advanced-cluster|oadp|kubevirt|kubernetes-nmstate)' | sort -u
```

## Planning Notes

- **Start the long ones early:** the LokiStack (12) and MultiCluster Observability (14) installs and the Windows image pipeline (17.1, ~31 min) progress in the background on their own. Started at the beginning of the day, they are ready when their turn comes. Download the Windows ISO (~5 GB) in advance.
- **Image signing (11):** an `ImagePolicy` starts a MachineConfig rollout on all nodes. On the test cluster it was only a CRI-O reload, no reboot (~1.5 min). Still, because the change goes to every node, a maintenance window is recommended on a live environment.
- **Shared resources:** 16 and 17 can affect some cluster-wide resources (DPA, `openshift-cnv`, `openshift-virtualization-os-images`). Those steps are marked separately in the guides.
- **Cleanup:** every guide ends with its own cleanup section.

## Conventions

- **"✅ Actual output"** lines are copied from the live test runs. Pod IPs, hostnames and timings are from the lab; yours will differ.
- **`REPLACE_ME_*`** placeholders in YAML files must be filled in for your environment (namespace, CIDR, node names, bucket names...). Each guide shows how, usually with a one-line `sed`.
- Lab network addresses are shortened as `<LAN>`, `<REMOTE>`, `<CORP_NET>`. Passwords in the files (`PocDemo2026x`, `app-pass-2026`...) are demo values — change them.
- All namespaces and users created by the guides start with `poc-` or `ocp-poc`, so they are easy to find and clean up.

## Languages

Every guide exists in English (`README.md`) and Turkish (`README.tr.md`) side by side in the same folder; YAML files and comments are in English and shared by both. Corrections and translations are welcome — please open an issue or a pull request.

## Disclaimer

This is an independent community project; it is not affiliated with or endorsed by Red Hat. The guides describe what happened on one lab cluster at one point in time. Read each step before running it, and don't run cluster-wide changes on a shared or production cluster without the platform owner's approval.

## License

[Apache License 2.0](LICENSE)
