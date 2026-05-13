# 40 — KubeConfig | Solutions

---

## Exercise 3 — Merged contexts

```bash
kubectl config get-contexts
```
```
CURRENT   NAME                          CLUSTER      AUTHINFO           NAMESPACE
*         kubernetes-admin@kubernetes   kubernetes   kubernetes-admin
          kubernetes-admin@cluster2     cluster2     kubernetes-admin
```

Both contexts are visible from both files merged via KUBECONFIG env var.

---

## Exercise 4 — dev-context

```bash
kubectl config get-contexts
```
```
CURRENT   NAME                          CLUSTER      AUTHINFO           NAMESPACE
          dev-context                   kubernetes   kubernetes-admin   kube-system
*         kubernetes-admin@kubernetes   kubernetes   kubernetes-admin
```

```bash
kubectl config use-context dev-context
kubectl get pods
# Lists kube-system pods without needing -n kube-system
```

---

## Exercise 7 — Decoded CA cert subject

```
Subject: CN = kubernetes
```

The CA cert subject is `kubernetes` — this is the cluster's root CA.

---

## Challenge Answers

**1. Three sections of kubeconfig?**
- `clusters`: API server addresses and CA certificates
- `users`: credentials (client certs, keys, tokens)
- `contexts`: named combinations of cluster + user + optional namespace

**2. current-context?**
It tells `kubectl` which context to use by default when no `--context` flag is given. It's the "active" context.

**3. Different kubeconfig file location?**
Two ways:
```bash
# Per command
kubectl get pods --kubeconfig=/path/to/config

# For the session
export KUBECONFIG=/path/to/config
```

**4. certificate-authority vs certificate-authority-data?**
- `certificate-authority`: path to a CA cert file on disk. File must exist on every machine using the kubeconfig.
- `certificate-authority-data`: base64-encoded CA cert embedded directly in the kubeconfig. Portable — works anywhere without needing the cert file separately.

**5. Can a context reference a different cluster's ClusterRole?**
No. A context links a user + cluster + namespace for connection purposes only. RBAC (ClusterRoles, RoleBindings) lives inside the target cluster and is independent of the kubeconfig. When you switch context, you connect to a different cluster with its own RBAC rules.
