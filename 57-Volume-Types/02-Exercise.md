# 57 — Volume Types | Exercises

> **Cluster:** Any kubectl-accessible cluster
> **Estimated time:** 30 minutes
> **Namespace:** `vol-lab`

---

## Exercise 1 — Setup

```bash
kubectl create namespace vol-lab
kubectl config set-context --current --namespace=vol-lab
```

---

## Exercise 2 — emptyDir: inter-container sharing

```yaml
# pod-emptydir.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-emptydir
  namespace: vol-lab
spec:
  containers:
  - name: writer
    image: busybox
    command: ["sh", "-c", "while true; do echo $(date) >> /shared/log.txt; sleep 2; done"]
    volumeMounts:
    - name: shared
      mountPath: /shared
  - name: reader
    image: busybox
    command: ["sh", "-c", "tail -f /shared/log.txt"]
    volumeMounts:
    - name: shared
      mountPath: /shared
  volumes:
  - name: shared
    emptyDir: {}
```

```bash
kubectl apply -f pod-emptydir.yaml
sleep 6
kubectl logs pod-emptydir -c reader
```

> **Observe:** reader sees timestamps written by writer — shared emptyDir.

---

## Exercise 3 — hostPath: read host filesystem

```yaml
# pod-hostpath.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-hostpath
  namespace: vol-lab
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "ls /host-tmp && sleep 3600"]
    volumeMounts:
    - name: host-tmp
      mountPath: /host-tmp
  volumes:
  - name: host-tmp
    hostPath:
      path: /tmp
      type: Directory
```

```bash
kubectl apply -f pod-hostpath.yaml
kubectl exec pod-hostpath -- ls /host-tmp | head -10
```

> **Observe:** You can see the node's /tmp directory from inside the container.

---

## Exercise 4 — configMap volume

```bash
kubectl create configmap app-config \
  --from-literal=database_host=db.example.com \
  --from-literal=log_level=INFO \
  -n vol-lab
```

```yaml
# pod-configmap-vol.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-configmap-vol
  namespace: vol-lab
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "ls /etc/config && cat /etc/config/database_host && sleep 3600"]
    volumeMounts:
    - name: config-vol
      mountPath: /etc/config
      readOnly: true
  volumes:
  - name: config-vol
    configMap:
      name: app-config
```

```bash
kubectl apply -f pod-configmap-vol.yaml
kubectl logs pod-configmap-vol
```

---

## Exercise 5 — secret volume

```bash
kubectl create secret generic db-secret \
  --from-literal=password=S3cr3t! \
  -n vol-lab
```

```yaml
# pod-secret-vol.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-secret-vol
  namespace: vol-lab
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "ls -la /etc/creds && cat /etc/creds/password && sleep 3600"]
    volumeMounts:
    - name: secret-vol
      mountPath: /etc/creds
      readOnly: true
  volumes:
  - name: secret-vol
    secret:
      secretName: db-secret
      defaultMode: 0400
```

```bash
kubectl apply -f pod-secret-vol.yaml
kubectl logs pod-secret-vol
kubectl exec pod-secret-vol -- ls -la /etc/creds
```

---

## Exercise 6 — Memory-backed emptyDir

```yaml
# pod-mem-emptydir.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-mem-emptydir
  namespace: vol-lab
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "df -h /ramdisk && sleep 3600"]
    volumeMounts:
    - name: ramdisk
      mountPath: /ramdisk
  volumes:
  - name: ramdisk
    emptyDir:
      medium: Memory
      sizeLimit: 64Mi
```

```bash
kubectl apply -f pod-mem-emptydir.yaml
kubectl logs pod-mem-emptydir
```

> **Observe:** Filesystem type is `tmpfs` — it's RAM-backed.

---

## Exercise 7 — Cleanup

```bash
kubectl delete namespace vol-lab
kubectl config set-context --current --namespace=default
```

---

## Challenge Questions

1. What is the difference between `emptyDir` and `hostPath`?
2. When would you use `emptyDir.medium: Memory`?
3. Why is `hostPath` considered a security risk?
4. How do ConfigMap volume mounts differ from environment variable injection?
5. What is the difference between `hostPath` and the `local` PV type?
