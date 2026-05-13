# 45 — Cluster Roles | Solutions

---

## Exercise 4 — Results

```bash
kubectl auth can-i get nodes --as michelle              # yes
kubectl auth can-i list persistentvolumes --as michelle # yes
kubectl auth can-i list pods --as michelle              # no
```

michelle can access cluster-scoped resources but not namespace-scoped ones — because we only granted node/PV access.

---

## Exercise 5 — Namespace-scoped ClusterRole binding

```bash
kubectl auth can-i create pods --as dev1 -n team-a   # yes
kubectl auth can-i create pods --as dev1 -n default  # no
kubectl auth can-i get nodes --as dev1               # no
```

Even though `edit` is a ClusterRole, binding it via RoleBinding restricts its scope to the `team-a` namespace. Node access (cluster-scoped) is not available even with the ClusterRole.

---

## Challenge Answers

**1. Can RoleBinding grant node access?**
No. Nodes are cluster-scoped (non-namespaced) resources. Only a ClusterRoleBinding can grant access to nodes. A RoleBinding can only grant permissions on namespaced resources within its namespace.

**2. ClusterRoleBinding vs RoleBinding with ClusterRole?**
- ClusterRoleBinding: grants ClusterRole permissions **across the entire cluster** including all namespaces and cluster-scoped resources.
- RoleBinding (referencing ClusterRole): grants ClusterRole permissions **only within the RoleBinding's namespace**. Cluster-scoped resource access in the ClusterRole is NOT granted.

**3. Read-only ClusterRole?**
The built-in `view` ClusterRole grants read-only access to most namespace-scoped resources. Bind it per-namespace with RoleBindings, or cluster-wide with ClusterRoleBinding.

**4. Admin access to one namespace only?**
```bash
kubectl create rolebinding user-admin \
  --clusterrole=admin \
  --user=username \
  -n target-namespace
```

**5. cluster-admin describe?**
```
PolicyRule:
  Resources  Non-Resource URLs  Verbs
  *.*        []                 [*]
             [*]                [*]
```
Wildcard `*` on resources, API groups, and verbs — full access to everything.
