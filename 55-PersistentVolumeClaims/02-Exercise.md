# 55 — Persistent Volume Claims | Exercises

> **Cluster:** Any kubectl-accessible cluster
> **Estimated time:** 25 minutes
> **Namespace:** `pvc-lab`

---

## Exercise 1 — Setup: Create a PV to bind against

```bash
kubectl create namespace pvc-lab
kubectl config set-context --current --namespace=pvc-lab
```

```yaml
# pv-for-pvc-lab.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-pvc-lab
spec:
  capacity:
    storage: 2Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /tmp/pv-pvc-lab
```

```bash
kubectl apply -f pv-for-pvc-lab.yaml
kubectl get pv pv-pvc-lab
```

---

## Exercise 2 — Create a PVC

```yaml
# my-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
  namespace: pvc-lab
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
  storageClassName: manual
```

```bash
kubectl apply -f my-pvc.yaml
kubectl get pvc
kubectl get pv pv-pvc-lab
```

> **Observe:** PVC STATUS should be `Bound`. PV should also show `Bound`.

---

## Exercise 3 — Create a pod that uses the PVC

```yaml
# pod-with-pvc.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-pvc
  namespace: pvc-lab
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "echo 'hello persistent world' > /data/message.txt && sleep 3600"]
    volumeMounts:
    - name: storage
      mountPath: /data
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: my-pvc
```

```bash
kubectl apply -f pod-with-pvc.yaml
kubectl exec pod-with-pvc -- cat /data/message.txt
```

---

## Exercise 4 — Verify data persists after pod restart

```bash
# Delete the pod
kubectl delete pod pod-with-pvc

# Recreate it
kubectl apply -f pod-with-pvc.yaml

# Check if the file still exists
kubectl exec pod-with-pvc -- cat /data/message.txt
```

> **Observe:** `hello persistent world` is still there — data persisted across pod restart.

---

## Exercise 5 — Create a Pending PVC (no matching PV)

```yaml
# pvc-pending.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-pending
  namespace: pvc-lab
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi     # no PV this large exists
  storageClassName: manual
```

```bash
kubectl apply -f pvc-pending.yaml
kubectl get pvc pvc-pending
kubectl describe pvc pvc-pending | grep -A3 Events
```

> **Observe:** PVC stays `Pending` with event: `no persistent volumes available`.

---

## Exercise 6 — PVC protection in action

```bash
# Try deleting the PVC while pod is running
kubectl delete pvc my-pvc &
kubectl get pvc my-pvc   # Shows Terminating, not deleted yet

# Delete the pod first
kubectl delete pod pod-with-pvc
kubectl get pvc my-pvc   # Now actually deleted
```

---

## Exercise 7 — Cleanup

```bash
kubectl delete pvc pvc-pending 2>/dev/null; true
kubectl delete pv pv-pvc-lab
kubectl delete namespace pvc-lab
kubectl config set-context --current --namespace=default
```

---

## Challenge Questions

1. What happens if you request more storage than any available PV provides?
2. Can two pods use the same PVC simultaneously?
3. What does the `selector` field in a PVC do?
4. How does Kubernetes choose between multiple matching PVs?
5. What is the `Lost` PVC phase and how does it happen?
