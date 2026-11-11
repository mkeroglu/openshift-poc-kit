# 01 — Authentication and Authorization (OAuth + RBAC)

> 🇹🇷 [Türkçe](README.tr.md) · [POC flow](../../README.md) · [02 — Container Management →](../02-container-management/README.md)

This guide shows how users log in to OpenShift with their **corporate identities** (LDAP/Active Directory) and how **role-based access control (RBAC)** limits who can do what, in which namespace.

Every step was **tested end to end on a live lab cluster** (OpenShift 4.22). The "✅ Actual output" lines below come from those tests. Environment-specific values (LDAP server, base DN, group names) must be adapted to your environment.

Scenario order:

1. Viewing the existing identity providers
2. LDAP / Active Directory identity provider
3. LDAP group synchronization
4. HTPasswd (local) identity provider
5. RBAC scenario: dev / prod, developer / viewer / namespace admin
6. Verification: permission matrix and real operations
7. Extra hardening (self-provisioner, kubeadmin)
8. Findings from the live test
9. Cleanup

---

## 1. Existing Identity Providers

```bash
oc get oauth cluster -o jsonpath='{range .spec.identityProviders[*]}{.name} ({.type}){"\n"}{end}'
oc get co authentication            # should be Available=True, Progressing=False, Degraded=False
oc get users; oc get identities     # users who have logged in, and which IdP they came from
oc get groups
```

- When a user logs in for the first time, OpenShift creates a **`User`** object and a linked **`Identity`** object (`<idp-name>:<user-id>`).
- A cluster can have several IdPs. Each one appears as a separate option on the login page.

**Console:** **Administration → Cluster Settings → Configuration → OAuth** → **Identity providers**.

---

## 2. LDAP / Active Directory Identity Provider

**Prerequisites:** the LDAP server address, a bind user for searching users (read-only is enough) and, if LDAPS is used, the CA of the server certificate.

```bash
# Bind user's password and the LDAP server's CA certificate
oc create secret generic ldap-bind-password -n openshift-config --from-literal=bindPassword='<BIND_PASSWORD>'
oc create configmap ldap-ca -n openshift-config --from-file=ca.crt=./ldap-ca.crt
```

`ldap-idp-patch.json` (adds the IdP **without removing** existing ones):

```json
[
  {
    "op": "add",
    "path": "/spec/identityProviders/-",
    "value": {
      "name": "Corporate-LDAP",
      "type": "LDAP",
      "mappingMethod": "claim",
      "ldap": {
        "url": "ldaps://<LDAP_SERVER>:636/<USER_BASE_DN>?sAMAccountName?sub?(objectClass=person)",
        "bindDN": "<BIND_USER_DN>",
        "bindPassword": { "name": "ldap-bind-password" },
        "ca": { "name": "ldap-ca" },
        "insecure": false,
        "attributes": {
          "id": ["sAMAccountName"],
          "preferredUsername": ["sAMAccountName"],
          "name": ["cn"],
          "email": ["mail"]
        }
      }
    }
  }
]
```

```bash
oc patch oauth cluster --type=json --patch-file ldap-idp-patch.json
oc get co authentication -w        # Progressing=True -> False (oauth pods roll out in ~1 min)
```

> - **For Active Directory** the username attribute is usually `sAMAccountName`. OpenLDAP uses `uid`.
> - The filter in the URL (`(objectClass=person)`) limits who can log in. For example, members of one group only: `(&(objectClass=person)(memberOf=<GROUP_DN>))`.
> - If a value is wrong, login returns `401 Unauthorized`. For details: `oc logs -n openshift-authentication deploy/oauth-openshift`.

✅ **Actual output:** In the lab, the LDAP (Active Directory) IdP is defined with `claim` mapping. The `User` object of a user who had logged in before showed `identities=["<LDAP-IdP-name>:<base64-user-id>"]`, confirming that the LDAP identity is linked to the user.

**Console:** **OAuth → Add → LDAP** form, or the **YAML** tab.

---

## 3. LDAP Group Synchronization

The LDAP IdP **only authenticates**; it does not bring LDAP groups into OpenShift. To grant permissions by LDAP group, the groups must be **synchronized**. There are two ways:

**a) `oc adm groups sync` (manually / from a CronJob)**

`ldap-sync.yaml`:

```yaml
kind: LDAPSyncConfig
apiVersion: v1
url: ldaps://<LDAP_SERVER>:636
bindDN: <BIND_USER_DN>
bindPassword: <BIND_PASSWORD>          # or: bindPassword: {file: /etc/secrets/bindPassword}
ca: ./ldap-ca.crt
augmentedActiveDirectory:
  groupsQuery:
    derefAliases: never
    pageSize: 0
  groupUIDAttribute: dn
  groupNameAttributes: [cn]
  usersQuery:
    baseDN: "<USER_BASE_DN>"
    scope: sub
    derefAliases: never
    filter: (objectClass=person)
    pageSize: 0
  userNameAttributes: [sAMAccountName]
  groupMembershipAttributes: ["memberOf:1.2.840.113556.1.4.1941:"]   # AD nested groups
```

```bash
oc adm groups sync --sync-config=ldap-sync.yaml --whitelist=groups.txt            # dry run (only shows)
oc adm groups sync --sync-config=ldap-sync.yaml --whitelist=groups.txt --confirm  # applies
```

`groups.txt` lists only the DNs of the groups that will be used in OpenShift. Syncing the whole AD creates thousands of unneeded groups.

**b) Group Sync Operator (continuous, scheduled)**

The **Group Sync Operator** from OperatorHub (`GroupSync` CR) does the same job on a schedule. In the lab, group sync runs with this operator.

✅ **Actual output:** In the lab, 251 LDAP groups are synchronized as OpenShift `Group` objects. One AD user was a member of 8 groups and received permissions through a `ClusterRoleBinding` bound to one of them.

---

## 4. HTPasswd (Local) Identity Provider

Used where there is no LDAP, for break-glass accounts or for test users.

```bash
htpasswd -c -B -b users.htpasswd poc-admin  '<PASSWORD>'
htpasswd    -B -b users.htpasswd poc-dev    '<PASSWORD>'
htpasswd    -B -b users.htpasswd poc-viewer '<PASSWORD>'
oc create secret generic poc-htpasswd -n openshift-config --from-file=htpasswd=users.htpasswd

oc patch oauth cluster --type=json --patch-file htpasswd-idp-patch.json   # ADDS to existing IdPs
oc get co authentication -w
```

To add or remove users, update `users.htpasswd` and re-apply the secret: `oc set data secret/poc-htpasswd -n openshift-config --from-file=htpasswd=users.htpasswd`.

> ⚠️ **If OAuth is managed by an ACM Governance Policy**, the added IdP is **removed again** shortly after.
>
> ✅ **Actual output:** In the lab, the patch returned `oauth.config.openshift.io/cluster patched`, but ~1 min later only the LDAP IdP was left. Cause: an ACM Policy in `enforce` mode with the `mustonlyhave` compliance type keeps resetting OAuth to the LDAP IdP only. A permanent change must be made **in the Policy itself** (or the Policy is temporarily switched to `inform`). That is why the RBAC tests below use **impersonation** instead of a real login. With impersonation, the API server makes the same authorization (RBAC) decision.
>
> To check: `oc get policies.policy.open-cluster-management.io -A | grep -i oauth`

---

## 5. RBAC Scenario

`rbac.yaml` creates two namespaces and three role profiles:

| User / Group | `poc-rbac-dev` | `poc-rbac-prod` |
|---|---|---|
| `poc-developers` (poc-dev) | `edit` (creates/changes everything) | `view` + custom `deployment-restarter` role |
| `poc-viewers` (poc-viewer) | `view` | `view` |
| `poc-admin` | `admin` | `admin` |

- **`view` / `edit` / `admin`** are built-in OpenShift ClusterRoles. A `RoleBinding` applies them **to that namespace only**.
- **`deployment-restarter`** is a namespace-scoped `Role`: only `get/list/watch/patch` on `deployments`. The developer can restart the app in prod (`oc rollout restart` is a patch) but cannot delete it or create new resources.
- The `view` role **does not show secrets**. `edit` does; `admin` can also grant permissions inside the namespace.
- The groups here are created by hand (`Group` objects). With LDAP sync, RoleBindings point directly at the synchronized LDAP groups (`name: <LDAP group name>`).

```bash
oc apply -f rbac.yaml
# An app and a secret in prod for testing
oc create deployment web --image=registry.access.redhat.com/ubi9/httpd-24 -n poc-rbac-prod
oc create secret generic db-pass --from-literal=password=x -n poc-rbac-prod
```

**Console:** **User Management → Groups** (groups and members), **User Management → RoleBindings → Create binding** (namespace, role, user/group). Permissions can also be granted from the **Project access** tab on the project page.

---

## 6. Verification

**With real users (recommended):** each user logs in with their own password (separate terminal or private browser window):

```bash
oc login <API_URL> -u poc-dev -p '<PASSWORD>'
oc auth can-i create deployments -n poc-rbac-prod
```

**With impersonation (admin, from one terminal):**

```bash
DEV="--as=poc-dev --as-group=poc-developers --as-group=system:authenticated"
oc auth can-i create deployments -n poc-rbac-dev $DEV
```

> ⚠️ With impersonation, the user's OpenShift `Group` memberships are **not resolved automatically**; the group must be passed explicitly with `--as-group`.
>
> ✅ **Actual output:** `oc auth can-i create deployments -n poc-rbac-dev --as=poc-dev` → **`no`**. The same command with `--as-group=poc-developers` → **`yes`**. On a real login, OpenShift adds the group membership itself.

**Permission matrix (`oc auth can-i ...`)** — ✅ **Actual output:**

| Operation | poc-dev | poc-viewer | poc-admin |
|---|---|---|---|
| dev: create deployment | yes | no | yes |
| prod: create deployment | **no** | no | yes |
| prod: restart deployment (patch) | **yes** | no | yes |
| prod: delete deployment | **no** | no | yes |
| prod: list pods | yes | yes | yes |
| prod: read secret | no | **no** | yes |
| prod: create rolebinding | no | no | **yes** |
| list pods in another namespace | no | no | **no** |
| list nodes (cluster scope) | no | no | **no** |

**Real operations** — ✅ **Actual output:**

```
[dev]    create deployment in dev  : deployment.apps/app1 created
[dev]    create deployment in prod : deployments.apps is forbidden: User "poc-dev" cannot create resource "deployments"
[dev]    restart in prod           : deployment.apps/web restarted
[dev]    delete in prod            : deployments.apps "web" is forbidden: User "poc-dev" cannot delete resource "deployments"
[viewer] read prod secret          : secrets "db-pass" is forbidden: User "poc-viewer" cannot get resource "secrets"
[viewer] list prod pods            : web-679bff89b5-klwf4 ContainerCreating  web-f97f9bf9d-k7n88 Running
[viewer] delete pod in prod        : pods "web-679bff89b5-klwf4" is forbidden: User "poc-viewer" cannot delete resource "pods"
[admin]  grant access in prod      : rolebinding.rbac.authorization.k8s.io/viewer-extra created
[admin]  make self cluster-admin   : clusterrolebindings.rbac.authorization.k8s.io is forbidden: User "poc-admin" cannot list resource "clusterrolebindings"
[admin]  list pods in another ns   : pods is forbidden: User "poc-admin" cannot list resource "pods"
```

The namespace admin can grant permissions in their own namespace, but cannot make themselves cluster-admin and cannot see other namespaces.

**Who can do what?**

```bash
oc adm policy who-can patch deployments -n poc-rbac-prod
```

✅ **Actual output:** `poc-admin` was listed among users, and `poc-developers` (plus corporate groups with cluster-admin) among groups.

---

## 7. Extra Hardening (cluster-wide — not applied in the POC)

**Stop users from creating their own projects** (enabled by default):

```bash
oc get clusterrolebinding self-provisioners -o jsonpath='{.subjects}'   # system:authenticated:oauth
oc patch clusterrolebinding.rbac self-provisioners -p '{"subjects": null}'
oc patch clusterrolebinding.rbac self-provisioners -p '{"metadata":{"annotations":{"rbac.authorization.kubernetes.io/autoupdate":"false"}}}'
```

After this, project creation is granted only to a specific group: `oc adm policy add-cluster-role-to-group self-provisioner <group>`.

**Removing the `kubeadmin` user:** after at least one user from the corporate IdP has been given `cluster-admin` and that login has been **verified**:

```bash
oc delete secret kubeadmin -n kube-system
```

> This cannot be undone. Keep the `kubeconfig` (system:admin) file somewhere safe for emergencies.

---

## 8. Findings From the Live Test

- **An ACM Policy rewrites OAuth:** see section 4. Before changing OAuth, check with `oc get policies.policy.open-cluster-management.io -A` whether a Policy manages it.
- **Impersonation does not resolve group membership:** see section 6.
- **Wrong / changed password:** LDAP login returns `Login failed (401 Unauthorized)`. Verify the password before retrying so the AD account does not get locked.

---

## 9. Cleanup

```bash
oc delete -f rbac.yaml
# If the HTPasswd IdP was added (check the index first):
oc get oauth cluster -o jsonpath='{range .spec.identityProviders[*]}{.name}{"\n"}{end}' | cat -n
oc patch oauth cluster --type=json -p '[{"op":"remove","path":"/spec/identityProviders/<INDEX>"}]'
oc delete secret poc-htpasswd -n openshift-config
oc delete user poc-admin poc-dev poc-viewer --ignore-not-found
oc get identity -o name | grep poc-local | xargs -r oc delete
```

---

## Summary

| Component | Role |
|---|---|
| `OAuth` (`cluster`) | The single object where identity providers (LDAP, HTPasswd, OIDC...) are defined |
| `User` / `Identity` | Created on first login; links the user to their identity in the IdP |
| `Group` | The unit permissions are granted to; synced from LDAP or created by hand |
| `ClusterRole` (`view`/`edit`/`admin`) | Built-in permission sets |
| `Role` | A namespace-scoped, fine-grained permission set (e.g. `deployment-restarter`) |
| `RoleBinding` | Grants a role to a user/group **in one namespace** |
| `ClusterRoleBinding` | Grants a role **cluster-wide** (use with care) |
