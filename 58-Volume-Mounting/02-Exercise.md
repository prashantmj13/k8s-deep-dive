# 58 — Volume Mounting | Exercises

> **Cluster:** Any kubectl-accessible cluster
> **Estimated time:** 25 minutes
> **Namespace:** `mount-lab`

---

## Exercise 1 — Setup

```bash
kubectl create namespace mount-lab
kubectl config set-context --current --namespace=mount-lab
```

---

## Exercise 2 — Multi-volume pod

Create a pod with 3 different volume types mounted simultaneously:

```bash
kubectl create configmap app-cfg \
  --from-literal=env=production \
  --from-literal=log_level=DEBUG \
  -n mount-lab

kubectl create secret generic app-secret \
  --from-literal=api_key=abc123 \
  -n mount-lab
```

```yaml
# pod-multi-vol.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-multi-vol
  namespace: mount-lab
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "
      echo '=== Config ===' &&
      ls /etc/config &&
      echo '=== Secret ===' &&
      ls /etc/secret &&
      echo '=== Temp ===' &&
      ls /tmp/work &&
      sleep 3600
    "]
    volumeMounts:
    - name: config
      mountPath: /etc/config
      readOnly: true
    - name: secret
      mountPath: /etc/secret
      readOnly: true
    - name: tmp
      mountPath: /tmp/work
  volumes:
  - name: config
    configMap:
      name: app-cfg
  - name: secret
    secret:
      secretName: app-secret
  - name: tmp
    emptyDir: {}
```

```bash
kubectl apply -f pod-multi-vol.yaml
kubectl logs pod-multi-vol
```

---

## Exercise 3 — subPath: inject a single file

```bash
kubectl create configmap nginx-cfg \
  --from-literal=nginx.conf="server { listen 80; }" \
  -n mount-lab
```

```yaml
# pod-subpath.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-subpath
  namespace: mount-lab
spec:
  containers:
  - name: nginx
    image: busybox
    command: ["sh", "-c", "cat /etc/nginx/nginx.conf && sleep 3600"]
    volumeMounts:
    - name: cfg
      mountPath: /etc/nginx/nginx.conf    # inject as a single file
      subPath: nginx.conf                  # key name from ConfigMap
  volumes:
  - name: cfg
    configMap:
      name: nginx-cfg
```

```bash
kubectl apply -f pod-subpath.yaml
kubectl logs pod-subpath
kubectl exec pod-subpath -- ls /etc/nginx
```

> **Observe:** Only `nginx.conf` appears at `/etc/nginx/nginx.conf` — the rest of `/etc/nginx/` from the image is untouched.

---

## Exercise 4 — Sidecar pattern with shared emptyDir

```yaml
# pod-sidecar.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-sidecar
  namespace: mount-lab
spec:
  containers:
  - name: writer
    image: busybox
    command: ["sh", "-c", "while true; do date >> /logs/app.log; sleep 3; done"]
    volumeMounts:
    - name: log-vol
      mountPath: /logs
  - name: reader
    image: busybox
    command: ["sh", "-c", "tail -f /logs/app.log"]
    volumeMounts:
    - name: log-vol
      mountPath: /logs
      readOnly: true
  volumes:
  - name: log-vol
    emptyDir: {}
```

```bash
kubectl apply -f pod-sidecar.yaml
sleep 10
kubectl logs pod-sidecar -c reader
```

---

## Exercise 5 — readOnly enforcement

```bash
# Verify secret volume is read-only
kubectl exec pod-multi-vol -- touch /etc/secret/newfile
```

> **Observe:** `Read-only file system` error — cannot write to readOnly mount.

```bash
# Temp volume IS writable
kubectl exec pod-multi-vol -- touch /tmp/work/newfile
kubectl exec pod-multi-vol -- ls /tmp/work
```

---

## Exercise 6 — Inspect mounts

```bash
kubectl describe pod pod-multi-vol | grep -A30 "Mounts:"
kubectl exec pod-multi-vol -- df -h
kubectl exec pod-multi-vol -- mount | grep -E "/etc/config|/etc/secret|/tmp/work"
```

---

## Exercise 7 — Cleanup

```bash
kubectl delete namespace mount-lab
kubectl config set-context --current --namespace=default
```

---

## Challenge Questions

1. What is `subPath` and when should you use it?
2. Why is it important to set `readOnly: true` on secret mounts?
3. Can two different containers in the same pod mount the same volume at different paths?
4. What happens if `mountPath` in the container already contains files from the image?
5. How do you verify what volumes are mounted inside a running container?
