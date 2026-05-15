# 54 — Persistent Volumes | Exercises

> **Cluster:** Any kubectl-accessible cluster (minikube or kubeadm)
> **Estimated time:** 20 minutes
> **Namespace:** `storage-lab`

---

## Exercise 1 — Setup

```bash
kubectl create namespace storage-lab
kubectl config set-context --current --namespace=storage-lab
```

---

## Exercise 2 — Create a hostPath PV

```yaml
# pv-hostpath.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-hostpath
  labels:
    type: local
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /mnt/pv-data
```

```bash
kubectl apply -f pv-hostpath.yaml
kubectl get pv
kubectl describe pv pv-hostpath
```

> **Observe:** Status should be `Available`.

---

## Exercise 3 — Create a second PV with different capacity

```yaml
# pv-large.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-large
spec:
  capacity:
    storage: 5Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: manual
  hostPath:
    path: /mnt/pv-large-data
```

```bash
kubectl apply -f pv-large.yaml
kubectl get pv
```

---

## Exercise 4 — Inspect PV details

```bash
# Check all PVs and their status
kubectl get pv -o wide

# Get PV as YAML
kubectl get pv pv-hostpath -o yaml | grep -E "capacity:|accessModes:|reclaimPolicy:|storageClassName:|status:"

# Check which PVs are available
kubectl get pv | grep Available
```

---

## Exercise 5 — Observe PV Phases

Create a PVC to bind to pv-hostpath (covered in detail in topic 55):

```yaml
# pvc-bind-test.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-bind-test
  namespace: storage-lab
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
  storageClassName: manual
```

```bash
kubectl apply -f pvc-bind-test.yaml
kubectl get pv pv-hostpath
```

> **Observe:** pv-hostpath status changes from `Available` → `Bound`.

---

## Exercise 6 — Release and observe

```bash
kubectl delete pvc pvc-bind-test
kubectl get pv pv-hostpath
```

> **Observe:** Status is now `Released` (not Available — because reclaimPolicy is Retain).
> To make it available again you must manually edit or delete and recreate the PV.

---

## Exercise 7 — Cleanup

```bash
kubectl delete pv pv-hostpath pv-large
kubectl delete namespace storage-lab
kubectl config set-context --current --namespace=default
```

---

## Challenge Questions

1. What is the difference between `Retain` and `Delete` reclaim policies?
2. Why does a `Released` PV not automatically become `Available`?
3. What access mode allows multiple nodes to read/write simultaneously?
4. Can a PV be in multiple namespaces?
5. What happens when you delete a PV that is still bound to a PVC?
