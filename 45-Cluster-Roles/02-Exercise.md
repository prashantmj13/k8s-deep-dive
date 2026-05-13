# 45 — Cluster Roles | Exercises

> **Cluster:** Any kubectl-accessible cluster
> **Estimated time:** 20 minutes

---

## Exercise 1 — List non-namespaced resources

```bash
kubectl api-resources --namespaced=false
```

These can only be accessed via ClusterRole.

---

## Exercise 2 — Inspect built-in ClusterRoles

```bash
kubectl describe clusterrole cluster-admin
kubectl describe clusterrole view
kubectl describe clusterrole edit
```

---

## Exercise 3 — Create a ClusterRole for nodes

```bash
kubectl create clusterrole node-viewer \
  --verb=get,list,watch \
  --resource=nodes,persistentvolumes

kubectl describe clusterrole node-viewer
```

---

## Exercise 4 — ClusterRoleBinding (cluster-wide access)

```bash
kubectl create clusterrolebinding michelle-node-viewer \
  --clusterrole=node-viewer \
  --user=michelle

kubectl auth can-i get nodes --as michelle          # yes
kubectl auth can-i list persistentvolumes --as michelle  # yes
kubectl auth can-i list pods --as michelle          # no
```

---

## Exercise 5 — RoleBinding with ClusterRole (namespace scoped)

```bash
kubectl create namespace team-a

# Bind built-in 'edit' ClusterRole to dev1 in team-a only
kubectl create rolebinding dev1-edit \
  --clusterrole=edit \
  --user=dev1 \
  -n team-a

kubectl auth can-i create pods --as dev1 -n team-a     # yes
kubectl auth can-i create pods --as dev1 -n default    # no
kubectl auth can-i get nodes --as dev1                 # no (ClusterRole scoped to namespace)
```

---

## Exercise 6 — Verify who has cluster-admin

```bash
kubectl get clusterrolebindings -o wide | grep cluster-admin
```

---

## Exercise 7 — Cleanup

```bash
kubectl delete clusterrole node-viewer
kubectl delete clusterrolebinding michelle-node-viewer
kubectl delete rolebinding dev1-edit -n team-a
kubectl delete namespace team-a
```

---

## Challenge Questions

1. Can a RoleBinding grant access to nodes?
2. What is the difference between binding a ClusterRole via ClusterRoleBinding vs RoleBinding?
3. Which ClusterRole gives full read-only access to all namespace resources?
4. How do you give a user admin access to one namespace only?
5. What happens when you `describe clusterrole cluster-admin`?
