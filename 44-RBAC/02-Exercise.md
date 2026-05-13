# 44 — RBAC | Exercises

> **Cluster:** Any kubectl-accessible cluster
> **Estimated time:** 25 minutes
> **Namespace:** `rbac-lab`

---

## Exercise 1 — Setup

```bash
kubectl create namespace rbac-lab
kubectl config set-context --current --namespace=rbac-lab
kubectl create deployment nginx --image=nginx --replicas=2
kubectl create configmap myconfig --from-literal=key=value
```

---

## Exercise 2 — Create a Role

Create a role that allows reading pods and listing configmaps:

```bash
kubectl create role pod-reader \
  --verb=get,list,watch \
  --resource=pods,configmaps \
  -n rbac-lab

kubectl describe role pod-reader -n rbac-lab
```

---

## Exercise 3 — Create a RoleBinding

Bind the role to user `jane`:

```bash
kubectl create rolebinding jane-pod-reader \
  --role=pod-reader \
  --user=jane \
  -n rbac-lab

kubectl describe rolebinding jane-pod-reader -n rbac-lab
```

---

## Exercise 4 — Test permissions with auth can-i

```bash
# Can jane list pods in rbac-lab?
kubectl auth can-i list pods -n rbac-lab --as jane

# Can jane delete pods in rbac-lab?
kubectl auth can-i delete pods -n rbac-lab --as jane

# Can jane list pods in default?
kubectl auth can-i list pods -n default --as jane

# Can jane get nodes?
kubectl auth can-i get nodes --as jane
```

---

## Exercise 5 — Create a ClusterRole for nodes

```bash
kubectl create clusterrole node-viewer \
  --verb=get,list,watch \
  --resource=nodes,persistentvolumes

kubectl create clusterrolebinding jane-node-viewer \
  --clusterrole=node-viewer \
  --user=jane

kubectl auth can-i get nodes --as jane
kubectl auth can-i list persistentvolumes --as jane
```

---

## Exercise 6 — Create RBAC for a ServiceAccount

```bash
kubectl create serviceaccount deploy-sa -n rbac-lab

kubectl create role deploy-manager \
  --verb=get,list,create,update,patch,delete \
  --resource=deployments \
  -n rbac-lab

kubectl create rolebinding deploy-sa-binding \
  --role=deploy-manager \
  --serviceaccount=rbac-lab:deploy-sa \
  -n rbac-lab

kubectl auth can-i list deployments \
  --as system:serviceaccount:rbac-lab:deploy-sa \
  -n rbac-lab
```

---

## Exercise 7 — View all permissions for a user

```bash
kubectl auth can-i --list --as jane -n rbac-lab
kubectl auth can-i --list --as jane
```

---

## Exercise 8 — Create RBAC declaratively

Create `dev-role.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: rbac-lab
rules:
- apiGroups: ["", "apps"]
  resources: ["pods", "deployments", "services"]
  verbs: ["*"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
  resourceNames: ["allowed-secret"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-team
  namespace: rbac-lab
subjects:
- kind: Group
  name: developers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f dev-role.yaml
kubectl auth can-i get secrets --as-group=developers --as=dev1 -n rbac-lab
```

---

## Exercise 9 — Cleanup

```bash
kubectl delete namespace rbac-lab
kubectl delete clusterrole node-viewer
kubectl delete clusterrolebinding jane-node-viewer
```

---

## Challenge Questions

1. What is the difference between a Role and a ClusterRole?
2. Can a RoleBinding reference a ClusterRole?
3. What does `kubectl auth can-i --list` show?
4. What `apiGroup` value do you use for core resources like pods and services?
5. How do you give a service account permission to access the API server?
