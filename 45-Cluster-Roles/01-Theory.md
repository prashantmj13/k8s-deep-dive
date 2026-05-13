# 45 — Cluster Roles and Cluster Role Bindings

## ClusterRole vs Role

| | Role | ClusterRole |
|-|------|-------------|
| Scope | Single namespace | Cluster-wide |
| Non-namespaced resources | ❌ No | ✅ Yes (nodes, PVs, namespaces) |
| Reusable across namespaces | ❌ No | ✅ Yes (via RoleBinding) |

---

## When to Use ClusterRole

Use a ClusterRole when you need to:
1. Grant access to **cluster-scoped resources**: nodes, persistentvolumes, namespaces, clusterroles, storageclasses
2. Grant access to **resources across all namespaces**
3. Create **reusable permission sets** bound to specific namespaces via RoleBindings

---

## Cluster-Scoped Resources

These resources have no namespace — they can only be granted via ClusterRole:

```bash
kubectl api-resources --namespaced=false
```

Output includes: `nodes`, `namespaces`, `persistentvolumes`, `storageclasses`, `clusterroles`, `clusterrolebindings`, `customresourcedefinitions`, `certificatesigningrequests`

---

## Creating a ClusterRole

```bash
# Imperative
kubectl create clusterrole node-admin \
  --verb=get,list,watch,update \
  --resource=nodes

# Declarative
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-admin
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch", "update", "patch"]
- apiGroups: [""]
  resources: ["persistentvolumes"]
  verbs: ["get", "list", "watch", "create", "delete"]
```

---

## ClusterRoleBinding — Cluster-wide

Grants permissions across the **entire cluster**:

```bash
kubectl create clusterrolebinding jane-node-admin \
  --clusterrole=node-admin \
  --user=jane
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jane-node-admin
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-admin
  apiGroup: rbac.authorization.k8s.io
```

---

## RoleBinding referencing a ClusterRole

Grants ClusterRole permissions **within a specific namespace only**:

```bash
kubectl create rolebinding jane-pod-admin \
  --clusterrole=admin \         # built-in ClusterRole
  --user=jane \
  -n production                 # scoped to production namespace only
```

This is a powerful pattern — define permissions once in a ClusterRole, scope them differently per namespace with RoleBindings.

---

## Built-in ClusterRoles

```bash
kubectl get clusterroles | grep -v system
```

| ClusterRole | Description |
|-------------|-------------|
| `cluster-admin` | Full access to everything |
| `admin` | Full namespace access (no RBAC/quota) |
| `edit` | Read/write in namespace |
| `view` | Read-only in namespace |
| `system:node` | Kubelet permissions |
| `system:kube-scheduler` | Scheduler permissions |
| `system:kube-controller-manager` | Controller manager permissions |

---

## Aggregate ClusterRoles

ClusterRoles can be composed using aggregation labels:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring
  labels:
    rbac.example.com/aggregate-to-monitoring: "true"
rules:
- apiGroups: [""]
  resources: ["pods", "nodes"]
  verbs: ["get", "list", "watch"]
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-aggregate
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.example.com/aggregate-to-monitoring: "true"
rules: []   # populated automatically by aggregation
```

---

## Useful Commands

```bash
# List all ClusterRoles
kubectl get clusterroles

# List all ClusterRoleBindings
kubectl get clusterrolebindings

# See who has cluster-admin
kubectl get clusterrolebindings -o wide | grep cluster-admin

# Describe a ClusterRole
kubectl describe clusterrole cluster-admin

# Check what a user can do cluster-wide
kubectl auth can-i --list --as jane
```
