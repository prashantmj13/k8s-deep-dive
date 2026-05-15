# 55 — Persistent Volume Claims (PVC)

## What is a PVC?

A **PersistentVolumeClaim** is a request for storage by a developer. It specifies how much storage is needed, what access mode is required, and optionally which StorageClass to use. Kubernetes finds a matching PV and binds them together.

Think of it like this:
- **PV** = an available apartment
- **PVC** = a rental application (requirements: 2 bed, 1 bath, pet-friendly)
- **Binding** = landlord approves — apartment is reserved

---

## PVC Spec

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
  namespace: default
spec:
  accessModes:
  - ReadWriteOnce           # must match PV's access mode
  resources:
    requests:
      storage: 5Gi          # minimum size needed
  storageClassName: manual  # must match PV's storageClassName
  volumeMode: Filesystem    # optional, default is Filesystem
  selector:                 # optional: target specific PVs by label
    matchLabels:
      type: ssd
```

---

## How Binding Works

Kubernetes finds a PV that satisfies ALL of:
1. `accessModes` — PV must support all requested modes
2. `storage` — PV capacity must be >= requested amount
3. `storageClassName` — must match exactly
4. `volumeMode` — must match
5. `selector` — PV labels must match (if specified)

**If multiple PVs match, Kubernetes picks the smallest sufficient one.**

If no match exists:
- PVC stays in `Pending` state
- Kubernetes keeps retrying until a matching PV appears

---

## PVC Phases

| Phase | Meaning |
|-------|---------|
| `Pending` | No matching PV found yet |
| `Bound` | Successfully bound to a PV |
| `Lost` | Bound PV was deleted — data may be lost |

---

## Using a PVC in a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: storage
      mountPath: /data        # where it appears inside the container
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: my-pvc       # references the PVC
```

---

## Dynamic Provisioning with StorageClass

With a StorageClass, you don't need a pre-created PV — it's created automatically:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard    # triggers dynamic provisioning
```

Kubernetes creates a PV automatically using the StorageClass provisioner.

---

## Expanding a PVC (Volume Expansion)

If the StorageClass supports expansion:

```bash
kubectl patch pvc my-pvc -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'
kubectl get pvc my-pvc
```

> The StorageClass must have `allowVolumeExpansion: true`.

---

## PVC Protection

Kubernetes adds a finalizer `kubernetes.io/pvc-protection` to any PVC in use by a pod. This prevents accidental deletion:

```bash
kubectl delete pvc my-pvc
# PVC enters Terminating state but is NOT deleted until the pod releases it
```

---

## Useful Commands

```bash
# List PVCs
kubectl get pvc
kubectl get pvc -A           # all namespaces

# Describe a PVC
kubectl describe pvc my-pvc

# Check which PV a PVC is bound to
kubectl get pvc my-pvc -o jsonpath='{.spec.volumeName}'

# Check which PVC a PV is bound to
kubectl get pv pv-demo -o jsonpath='{.spec.claimRef.name}'

# Watch PVC status
kubectl get pvc -w
```

---

## Static vs Dynamic Provisioning

| | Static | Dynamic |
|-|--------|---------|
| PV created by | Admin manually | StorageClass automatically |
| PVC specifies | `storageClassName: manual` (or empty) | `storageClassName: <class>` |
| Flexibility | Fixed, must pre-plan | On-demand |
| Best for | On-prem, NFS | Cloud environments |
