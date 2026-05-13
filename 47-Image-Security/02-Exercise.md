# 47 — Image Security | Exercises

> **Cluster:** Any kubectl-accessible cluster
> **Estimated time:** 20 minutes

---

## Exercise 1 — Create an imagePullSecret

```bash
kubectl create secret docker-registry my-creds \
  --docker-server=registry.example.com \
  --docker-username=testuser \
  --docker-password=testpass \
  --docker-email=test@example.com

kubectl get secret my-creds
kubectl describe secret my-creds
```

---

## Exercise 2 — Use imagePullSecret in a pod

```yaml
# pod-private.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-private
spec:
  imagePullSecrets:
  - name: my-creds
  containers:
  - name: app
    image: registry.example.com/myapp:v1.0
```

```bash
kubectl apply -f pod-private.yaml
kubectl describe pod pod-private | grep -E "Image:|imagePullSecrets"
```

(Pod will fail to pull since the registry is fake — that's expected)

---

## Exercise 3 — Attach imagePullSecret to default SA

```bash
kubectl patch serviceaccount default \
  -p '{"imagePullSecrets": [{"name": "my-creds"}]}'

kubectl get serviceaccount default -o yaml | grep imagePullSecrets -A3
```

---

## Exercise 4 — Pod security context (non-root)

```yaml
# pod-secure.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-secure
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
  containers:
  - name: app
    image: nginx
    securityContext:
      readOnlyRootFilesystem: false   # nginx needs write access
      allowPrivilegeEscalation: false
```

```bash
kubectl apply -f pod-secure.yaml
kubectl exec pod-secure -- id
kubectl exec pod-secure -- whoami
```

---

## Exercise 5 — Check image pull policy

```bash
kubectl run latest-test --image=nginx:latest
kubectl describe pod latest-test | grep "Image Pull Policy"

kubectl run pinned-test --image=nginx:1.25.3
kubectl describe pod pinned-test | grep "Image Pull Policy"
```

> Default: `Always` for `:latest`, `IfNotPresent` for other tags.

---

## Exercise 6 — Cleanup

```bash
kubectl delete pod pod-private pod-secure latest-test pinned-test 2>/dev/null; true
kubectl delete secret my-creds
kubectl patch serviceaccount default -p '{"imagePullSecrets": null}'
```

---

## Challenge Questions

1. What secret type is used for registry credentials?
2. What is the difference between `imagePullPolicy: Always`, `IfNotPresent`, and `Never`?
3. Why should you never use `:latest` tag in production?
4. How do you make an imagePullSecret available to all pods in a namespace without adding it to every pod spec?
5. What does `runAsNonRoot: true` do if the image runs as root?
