# 40 — KubeConfig | Exercises

> **Cluster:** Any kubectl-accessible cluster
> **Estimated time:** 20 minutes

---

## Exercise 1 — Inspect current kubeconfig

```bash
kubectl config view
kubectl config current-context
kubectl config get-contexts
kubectl config get-clusters
kubectl config get-users
```

---

## Exercise 2 — Create a second kubeconfig file

```bash
# Copy current config as a second cluster config
cp ~/.kube/config /tmp/cluster2.kubeconfig

# Edit it to simulate a second cluster (just change the name)
sed -i 's/kubernetes/cluster2/g' /tmp/cluster2.kubeconfig

cat /tmp/cluster2.kubeconfig | grep -E "name:|server:"
```

---

## Exercise 3 — Merge two kubeconfig files

```bash
export KUBECONFIG=~/.kube/config:/tmp/cluster2.kubeconfig
kubectl config get-contexts
```

> You should now see contexts from both files.

---

## Exercise 4 — Add a new context with a different namespace

```bash
kubectl config set-context dev-context \
  --cluster=kubernetes \
  --user=kubernetes-admin \
  --namespace=kube-system

kubectl config get-contexts
kubectl config use-context dev-context
kubectl get pods   # should list kube-system pods without -n flag
```

---

## Exercise 5 — Switch contexts and verify

```bash
kubectl config use-context kubernetes-admin@kubernetes
kubectl config current-context

kubectl config use-context dev-context
kubectl config current-context
kubectl get pods   # shows kube-system pods (default ns is kube-system)
```

---

## Exercise 6 — Set default namespace on current context

```bash
kubectl config use-context kubernetes-admin@kubernetes
kubectl config set-context --current --namespace=kube-system
kubectl get pods   # no -n needed, shows kube-system pods

# Reset to default
kubectl config set-context --current --namespace=default
```

---

## Exercise 7 — View raw (decoded) kubeconfig

```bash
kubectl config view --raw | grep certificate-authority-data | \
  awk '{print $2}' | base64 -d | openssl x509 -noout -subject
```

---

## Exercise 8 — Cleanup

```bash
kubectl config delete-context dev-context
unset KUBECONFIG
```

---

## Challenge Questions

1. What are the three sections of a kubeconfig file?
2. What is `current-context` used for?
3. How do you use a kubeconfig file that's not at `~/.kube/config`?
4. What is the difference between `certificate-authority` and `certificate-authority-data`?
5. Can one context reference a ClusterRole from a different cluster entry?
