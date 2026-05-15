# 54 — Persistent Volumes (PV)

## The Problem with Pod Storage

By default, container storage is **ephemeral** — when a pod dies, all its data is lost. For stateful applications (databases, file storage, message queues), you need storage that **outlives any individual pod**.

---

## Kubernetes Storage Architecture

```
Developer creates PVC (what they need)
         ↓
Kubernetes binds PVC to a matching PV (what exists)
         ↓
Pod mounts the PVC as a volume
         ↓
Container reads/writes to the volume
         ↓
Pod dies — data stays in PV
```

---

## What is a PersistentVolume (PV)?

A PersistentVolume is a **cluster-level storage resource** provisioned by an administrator (or dynamically by a StorageClass). It represents actual storage — a disk on a cloud provider, an NFS share, a local directory, etc.

PVs are **independent of any pod or namespace** — they exist at the cluster level.

---

## PV Spec — Key Fields

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-demo
spec:
  capacity:
    storage: 10Gi              # size of the volume
  volumeMode: Filesystem       # Filesystem or Block
  accessModes:
  - ReadWriteOnce              # how it can be mounted
  persistentVolumeReclaimPolicy: Retain   # what happens after PVC is deleted
  storageClassName: manual     # links to a StorageClass (or "" for static)
  hostPath:                    # the actual storage backend
    path: /mnt/data
```

---

## Access Modes

| Mode | Abbreviation | Meaning |
|------|-------------|---------|
| `ReadWriteOnce` | RWO | One node can mount read/write |
| `ReadOnlyMany` | ROX | Many nodes can mount read-only |
| `ReadWriteMany` | RWX | Many nodes can mount read/write |
| `ReadWriteOncePod` | RWOP | One pod can mount read/write (K8s 1.22+) |

> Not all backends support all modes. Check your storage provider's docs.

| Backend | RWO | ROX | RWX |
|---------|-----|-----|-----|
| AWS EBS | ✅ | ✅ | ❌ |
| GCP PD | ✅ | ✅ | ❌ |
| Azure Disk | ✅ | ❌ | ❌ |
| NFS | ✅ | ✅ | ✅ |
| CephFS | ✅ | ✅ | ✅ |

---

## Reclaim Policies

What happens to the PV after the PVC that bound it is deleted:

| Policy | Behaviour |
|--------|-----------|
| `Retain` | PV stays, data preserved. Manual cleanup required. |
| `Delete` | PV and underlying storage are deleted automatically. |
| `Recycle` | (deprecated) Scrubs the volume and makes it available again. |

> **Production recommendation:** Use `Retain` for critical data. Use `Delete` for ephemeral workloads with dynamic provisioning.

---

## PV Phases (Lifecycle)

```
Available → Bound → Released → (Failed)
```

| Phase | Meaning |
|-------|---------|
| `Available` | Free, not yet bound to a PVC |
| `Bound` | Bound to a PVC |
| `Released` | PVC deleted, but PV not yet reclaimed |
| `Failed` | Automatic reclamation failed |

---

## Static Provisioning Example

Admin creates the PV manually:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nfs-01
  labels:
    type: nfs
spec:
  capacity:
    storage: 20Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""          # empty = no StorageClass (static)
  nfs:
    server: 192.168.1.100
    path: /exports/data
```

---

## Useful Commands

```bash
# List all PVs
kubectl get pv
kubectl get pv -o wide

# Describe a PV
kubectl describe pv pv-demo

# Check PV status
kubectl get pv | awk '{print $1, $2, $5, $6}'

# Delete a PV (only after PVC is deleted and data backed up)
kubectl delete pv pv-demo
```

---

## PV vs PVC vs StorageClass Summary

| Object | Who creates it | Purpose |
|--------|---------------|---------|
| PV | Admin (static) or StorageClass (dynamic) | Actual storage resource |
| PVC | Developer | Request for storage |
| StorageClass | Admin | Template for dynamic PV creation |
