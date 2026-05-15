# 54 — Persistent Volumes | Solutions

---

## Exercise 2 — PV created

```bash
kubectl get pv
```
```
NAME          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      STORAGECLASS
pv-hostpath   1Gi        RWO            Retain           Available   manual
pv-large      5Gi        RWO            Delete           Available   manual
```

Both PVs are `Available` — waiting to be claimed.

---

## Exercise 5 — After PVC binds

```bash
kubectl get pv pv-hostpath
```
```
NAME          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                        STORAGECLASS
pv-hostpath   1Gi        RWO            Retain           Bound    storage-lab/pvc-bind-test    manual
```

`STATUS` changed to `Bound` and `CLAIM` column shows which PVC is using it.

> Note: `pv-large` (5Gi) was NOT bound even though it also matches the access mode. Kubernetes chose `pv-hostpath` (1Gi) because it's a closer match in size to the 500Mi request — smallest sufficient PV wins.

---

## Exercise 6 — After deleting PVC

```bash
kubectl get pv pv-hostpath
```
```
NAME          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     STORAGECLASS
pv-hostpath   1Gi        RWO            Retain           Released   manual
```

`Released` — PVC is gone but data is preserved. The PV is NOT automatically available because Kubernetes keeps the `claimRef` on the PV to prevent accidental reuse.

To manually make it available again:
```bash
kubectl patch pv pv-hostpath -p '{"spec":{"claimRef": null}}'
kubectl get pv pv-hostpath   # STATUS: Available
```

---

## Challenge Answers

**1. Retain vs Delete reclaim policy?**
- `Retain`: PV stays after PVC deletion. Data is preserved. Admin must manually inspect, backup, and clean up. The PV goes to `Released` state and won't be rebound automatically. Use for critical data.
- `Delete`: PV and underlying storage (cloud disk, NFS export etc.) are automatically deleted when the PVC is deleted. Storage is freed immediately. Use for ephemeral/replaceable workloads with dynamic provisioning.

**2. Released PV doesn't become Available?**
Kubernetes leaves `claimRef` (reference to the old PVC) on the PV even after the PVC is deleted. This prevents a new PVC from accidentally binding to a PV that may contain sensitive data from the old PVC. An admin must explicitly clear the `claimRef` to make the PV available again.

**3. Multiple nodes read/write simultaneously?**
`ReadWriteMany` (RWX). Supported by NFS, CephFS, Azure File, and some other distributed storage backends. NOT supported by cloud block storage like AWS EBS or GCP Persistent Disk.

**4. Can a PV be in multiple namespaces?**
No. PVs are cluster-scoped resources (not namespaced). However, a PV can only be bound to ONE PVC at a time. Once bound, it is effectively dedicated to the namespace of that PVC until released.

**5. Deleting a bound PV?**
The PV enters `Terminating` state but is NOT immediately deleted — it waits until the PVC is deleted first (a finalizer `kubernetes.io/pv-protection` prevents deletion). This protects running pods from losing their storage unexpectedly.
