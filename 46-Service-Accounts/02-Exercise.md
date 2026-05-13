# 46 — Service Accounts | Exercises

> **Cluster:** Any kubectl-accessible cluster
> **Estimated time:** 20 minutes
> **Namespace:** `sa-lab`

---

## Exercise 1 — Inspect the default service account

```bash
kubectl create namespace sa-lab
kubectl config set-context --current --namespace=sa-lab

kubectl get serviceaccounts
kubectl describe serviceaccount default
```

---

## Exercise 2 — Create a custom service account

```bash
kubectl create serviceaccount dashboard-sa
kubectl get serviceaccounts
kubectl describe serviceaccount dashboard-sa
```

---

## Exercise 3 — Run a pod with the service account

```yaml
# pod-with-sa.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-sa
  namespace: sa-lab
spec:
  serviceAccountName: dashboard-sa
  containers:
  - name: app
    image: nginx
```

```bash
kubectl apply -f pod-with-sa.yaml
kubectl exec pod-with-sa -- ls /var/run/secrets/kubernetes.io/serviceaccount/
kubectl exec pod-with-sa -- cat /var/run/secrets/kubernetes.io/serviceaccount/namespace
```

---

## Exercise 4 — Read the token and decode it

```bash
TOKEN=$(kubectl exec pod-with-sa -- cat /var/run/secrets/kubernetes.io/serviceaccount/token)
echo $TOKEN | cut -d. -f2 | base64 -d 2>/dev/null | python3 -m json.tool 2>/dev/null || \
echo $TOKEN | cut -d. -f2 | base64 -d 2>/dev/null
```

Note the `sub` field — it shows the service account identity.

---

## Exercise 5 — Grant the SA API permissions

```bash
kubectl create role pod-reader \
  --verb=get,list \
  --resource=pods \
  -n sa-lab

kubectl create rolebinding dashboard-sa-reader \
  --role=pod-reader \
  --serviceaccount=sa-lab:dashboard-sa \
  -n sa-lab

kubectl auth can-i list pods \
  --as system:serviceaccount:sa-lab:dashboard-sa \
  -n sa-lab
```

---

## Exercise 6 — Create a long-lived token

```bash
kubectl create token dashboard-sa --duration=24h -n sa-lab
```

---

## Exercise 7 — Disable auto-mount

```yaml
# pod-no-token.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-no-token
  namespace: sa-lab
spec:
  automountServiceAccountToken: false
  containers:
  - name: app
    image: nginx
```

```bash
kubectl apply -f pod-no-token.yaml
kubectl exec pod-no-token -- ls /var/run/secrets/ 2>&1
```

---

## Exercise 8 — Cleanup

```bash
kubectl delete namespace sa-lab
kubectl config set-context --current --namespace=default
```

---

## Challenge Questions

1. What three files are auto-mounted in every pod's service account volume?
2. How do you prevent a pod from having API server access at all?
3. What changed in Kubernetes v1.24 regarding service account tokens?
4. How is a service account referenced in an RBAC subject?
5. Can two pods in different namespaces share the same service account?
