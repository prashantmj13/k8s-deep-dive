# 44 — Role Based Access Controls (RBAC)

## What is RBAC?

RBAC controls **who** (subject) can perform **what actions** (verbs) on **which resources** in Kubernetes. It is the recommended authorization mechanism.

---

## RBAC Objects

| Object | Scope | Purpose |
|--------|-------|---------|
| `Role` | Namespace | Permissions within one namespace |
| `ClusterRole` | Cluster-wide | Permissions across all namespaces or non-namespaced resources |
| `RoleBinding` | Namespace | Binds a Role/ClusterRole to a subject within a namespace |
| `ClusterRoleBinding` | Cluster-wide | Binds a ClusterRole to a subject across all namespaces |

---

## Role — Namespace Scoped

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
- apiGroups: [""]          # "" means core API group
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list"]
```

### Common Verbs

| Verb | HTTP Method |
|------|-------------|
| `get` | GET (single resource) |
| `list` | GET (collection) |
| `watch` | GET with watch parameter |
| `create` | POST |
| `update` | PUT |
| `patch` | PATCH |
| `delete` | DELETE |
| `deletecollection` | DELETE (collection) |

---

## RoleBinding — Bind Role to Subject

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jane-pod-reader
  namespace: default
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

Subjects can be:
- `User` — a human user (authenticated via cert/token)
- `Group` — a group of users
- `ServiceAccount` — a pod's service account

---

## ClusterRole — Cluster-wide Permissions

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["persistentvolumes"]
  verbs: ["get", "list"]
```

---

## ClusterRoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jane-node-reader
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## Imperative RBAC Commands

```bash
# Create role
kubectl create role pod-reader \
  --verb=get,list,watch \
  --resource=pods \
  -n default

# Create role binding
kubectl create rolebinding jane-pod-reader \
  --role=pod-reader \
  --user=jane \
  -n default

# Create cluster role
kubectl create clusterrole node-reader \
  --verb=get,list,watch \
  --resource=nodes

# Create cluster role binding
kubectl create clusterrolebinding jane-node-reader \
  --clusterrole=node-reader \
  --user=jane
```

---

## Check Permissions — auth can-i

```bash
# Can I create pods?
kubectl auth can-i create pods

# Can jane list deployments in kube-system?
kubectl auth can-i list deployments -n kube-system --as jane

# Can service account myapp-sa get secrets?
kubectl auth can-i get secrets --as system:serviceaccount:default:myapp-sa

# List all actions I can do
kubectl auth can-i --list
kubectl auth can-i --list --as jane
```

---

## Resource Names — Fine-grained Access

Restrict access to specific named resources:

```yaml
rules:
- apiGroups: [""]
  resources: ["pods"]
  resourceNames: ["my-pod", "specific-pod"]
  verbs: ["get", "delete"]
```

---

## API Groups

| API Group | Resources |
|-----------|-----------|
| `""` (core) | pods, services, configmaps, secrets, nodes |
| `apps` | deployments, replicasets, daemonsets, statefulsets |
| `batch` | jobs, cronjobs |
| `autoscaling` | horizontalpodautoscalers |
| `rbac.authorization.k8s.io` | roles, rolebindings, clusterroles |
| `networking.k8s.io` | ingresses, networkpolicies |

---

## Built-in ClusterRoles

| Role | Description |
|------|-------------|
| `cluster-admin` | Full access to everything |
| `admin` | Full access within a namespace |
| `edit` | Read/write in a namespace (no RBAC) |
| `view` | Read-only in a namespace |
| `system:node` | Kubelet permissions |
