# 48 — Security Contexts | Exercises

> **Cluster:** Any kubectl-accessible cluster
> **Estimated time:** 20 minutes

---

## Exercise 1 — Run as specific user/group

```yaml
# pod-user.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-user
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "id && sleep 3600"]
```

```bash
kubectl apply -f pod-user.yaml
kubectl logs pod-user
kubectl exec pod-user -- id
```

---

## Exercise 2 — Prevent privilege escalation

```yaml
# pod-noprivesc.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-noprivesc
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    securityContext:
      allowPrivilegeEscalation: false
      runAsNonRoot: true
      runAsUser: 1000
```

```bash
kubectl apply -f pod-noprivesc.yaml
kubectl exec pod-noprivesc -- id
```

---

## Exercise 3 — Read-only root filesystem

```yaml
# pod-readonly.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-readonly
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    securityContext:
      readOnlyRootFilesystem: true
    volumeMounts:
    - name: tmp
      mountPath: /tmp
  volumes:
  - name: tmp
    emptyDir: {}
```

```bash
kubectl apply -f pod-readonly.yaml

# This should FAIL (root is read-only):
kubectl exec pod-readonly -- touch /newfile

# This should SUCCEED (tmp is writable emptyDir):
kubectl exec pod-readonly -- touch /tmp/newfile
```

---

## Exercise 4 — Linux capabilities

```yaml
# pod-caps.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-caps
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    securityContext:
      capabilities:
        drop: ["ALL"]
        add: ["NET_ADMIN"]
```

```bash
kubectl apply -f pod-caps.yaml
kubectl exec pod-caps -- cat /proc/1/status | grep Cap
```

---

## Exercise 5 — fsGroup volume ownership

```yaml
# pod-fsgroup.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-fsgroup
spec:
  securityContext:
    runAsUser: 1000
    fsGroup: 2000
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "ls -la /data && sleep 3600"]
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    emptyDir: {}
```

```bash
kubectl apply -f pod-fsgroup.yaml
kubectl logs pod-fsgroup
kubectl exec pod-fsgroup -- ls -la /
```

---

## Exercise 6 — Cleanup

```bash
kubectl delete pod pod-user pod-noprivesc pod-readonly pod-caps pod-fsgroup
```

---

## Challenge Questions

1. What is the difference between pod-level and container-level security context?
2. What does `allowPrivilegeEscalation: false` prevent?
3. What is `fsGroup` used for?
4. Why is `readOnlyRootFilesystem: true` a security best practice?
5. What is the safest capabilities approach for production containers?
