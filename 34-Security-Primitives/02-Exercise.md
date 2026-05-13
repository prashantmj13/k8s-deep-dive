# 34 — Security Primitives | Exercises

> **Cluster:** Any kubectl-accessible cluster
> **Estimated time:** 15 minutes

---

## Exercise 1 — Inspect API server security settings

```bash
kubectl describe pod kube-apiserver-controlplane -n kube-system | grep -E "authorization-mode|admission-plugins|client-ca"
```

Note what authorization modes and admission plugins are enabled.

---

## Exercise 2 — Check current auth mode in API server manifest

```bash
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep -E "authorization|admission"
```

---

## Exercise 3 — List all enabled admission plugins

```bash
kubectl exec -n kube-system kube-apiserver-controlplane -- \
  kube-apiserver --help 2>&1 | grep "enable-admission-plugins"
```

---

## Exercise 4 — Check TLS certificates in use

```bash
kubectl describe pod kube-apiserver-controlplane -n kube-system | grep -E "tls|cert|ca"
```

---

## Exercise 5 — Check who has cluster-admin rights

```bash
kubectl get clusterrolebindings -o wide | grep cluster-admin
```

---

## Exercise 6 — Check if audit logging is enabled

```bash
grep -i audit /etc/kubernetes/manifests/kube-apiserver.yaml
```

---

## Challenge Questions

1. What are the three phases a request goes through before hitting etcd?
2. Which authorization mode is recommended for production and why?
3. What is the difference between a mutating and validating admission controller?
4. Why is TLS required between all cluster components?
5. What does `--authorization-mode=Node,RBAC` mean?
