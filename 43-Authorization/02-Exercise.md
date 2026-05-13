# 43 — Authorization | Exercises

> **Cluster:** Any kubectl-accessible cluster
> **Estimated time:** 15 minutes

---

## Exercise 1 — Check current authorization mode

```bash
kubectl describe pod kube-apiserver-controlplane -n kube-system | grep authorization-mode
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep authorization-mode
```

---

## Exercise 2 — Test authorization with auth can-i

```bash
# As yourself (admin)
kubectl auth can-i create pods
kubectl auth can-i delete nodes
kubectl auth can-i "*" "*"   # can I do everything?

# As jane (no RBAC yet)
kubectl auth can-i create pods --as jane
kubectl auth can-i list pods --as jane -n default
kubectl auth can-i get nodes --as jane
```

---

## Exercise 3 — Create a limited user and verify

```bash
# Create a role and binding for jane
kubectl create role jane-role --verb=get,list --resource=pods -n default
kubectl create rolebinding jane-bind --role=jane-role --user=jane -n default

# Now test
kubectl auth can-i list pods --as jane -n default    # yes
kubectl auth can-i create pods --as jane -n default  # no
kubectl auth can-i list pods --as jane -n kube-system # no
```

---

## Exercise 4 — SubjectAccessReview

```bash
kubectl create -f - <<EOF
apiVersion: authorization.k8s.io/v1
kind: SubjectAccessReview
spec:
  user: jane
  resourceAttributes:
    verb: list
    resource: pods
    namespace: default
EOF
```

Look for `status.allowed: true` in the output.

---

## Exercise 5 — Check Node authorization

```bash
# Kubelet credential uses system:node group
kubectl get csr | head -5
# On a running cluster, node CSRs use system:node:<name>

# Check kubelet config
cat /var/lib/kubelet/config.yaml | grep -i auth
```

---

## Challenge Questions

1. What does `--authorization-mode=Node,RBAC` mean in practice?
2. What is the difference between authorization and admission control?
3. What command quickly checks if a user is allowed to perform an action?
4. Why is RBAC "deny by default"?
5. When would you use Webhook authorization instead of RBAC?
