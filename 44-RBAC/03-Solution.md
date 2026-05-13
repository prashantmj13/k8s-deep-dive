# 44 — RBAC | Solutions

---

## Exercise 4 — auth can-i results

```bash
kubectl auth can-i list pods -n rbac-lab --as jane
# yes

kubectl auth can-i delete pods -n rbac-lab --as jane
# no

kubectl auth can-i list pods -n default --as jane
# no   ← binding is only for rbac-lab namespace

kubectl auth can-i get nodes --as jane
# no   ← no cluster role yet
```

---

## Exercise 5 — After ClusterRole

```bash
kubectl auth can-i get nodes --as jane
# yes

kubectl auth can-i list persistentvolumes --as jane
# yes
```

---

## Exercise 6 — Service account permissions

```bash
kubectl auth can-i list deployments \
  --as system:serviceaccount:rbac-lab:deploy-sa \
  -n rbac-lab
# yes
```

Note: Service account format is always `system:serviceaccount:<namespace>:<name>`

---

## Exercise 7 — List all permissions for jane

```
Resources                Non-Resource URLs   Resource Names   Verbs
pods                     []                  []               [get list watch]
configmaps               []                  []               [get list watch]
nodes                    []                  []               [get list watch]
persistentvolumes        []                  []               [get list watch]
```

---

## Challenge Answers

**1. Role vs ClusterRole?**
- `Role`: scoped to a single namespace. Cannot grant access to cluster-wide resources (nodes, PVs).
- `ClusterRole`: applies across all namespaces and can also grant access to non-namespaced resources (nodes, PVs, CRDs). A ClusterRole can be bound within a namespace using a RoleBinding.

**2. Can a RoleBinding reference a ClusterRole?**
Yes! This is a common pattern. A ClusterRole defines reusable permissions; a RoleBinding restricts them to one namespace. Example: create a ClusterRole `pod-viewer` once, then use RoleBindings in different namespaces to grant it to different teams — without creating duplicate Roles.

**3. kubectl auth can-i --list?**
Lists all resources and verbs the current user (or `--as` user) has permission to access, grouped by resource. Very useful for auditing and debugging permission issues.

**4. apiGroup for core resources?**
Use `""` (empty string). Core resources (pods, services, configmaps, secrets, nodes) belong to the core API group which has no name. Non-core resources use their group name: `apps`, `batch`, `networking.k8s.io`, etc.

**5. Grant service account API access?**
Create a Role or ClusterRole with the desired permissions, then create a RoleBinding/ClusterRoleBinding with the subject:
```yaml
subjects:
- kind: ServiceAccount
  name: my-sa
  namespace: my-namespace
```
Or imperatively:
```bash
kubectl create rolebinding sa-binding \
  --role=my-role \
  --serviceaccount=namespace:sa-name
```
