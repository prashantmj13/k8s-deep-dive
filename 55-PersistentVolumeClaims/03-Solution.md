# 55 — Persistent Volume Claims | Solutions

---

## Exercise 2 — After PVC creation

```bash
kubectl get pvc
```
```
NAME     STATUS   VOLUME       CAPACITY   ACCESS MODES   STORAGECLASS   AGE
my-pvc   Bound    pv-pvc-lab   2Gi        RWO            manual         5s
```

```bash
kubectl get pv pv-pvc-lab
```
```
NAME         CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM              STORAGECLASS
pv-pvc-lab   2Gi        RWO            Retain           Bound    pvc-lab/my-pvc     manual
```

> Note: PVC requested 500Mi but got bound to a 2Gi PV (smallest available match). The CAPACITY shown is the PV's full capacity, not the requested amount.

---

## Exercise 3 — Pod writes data

```bash
kubectl exec pod-with-pvc -- cat /data/message.txt
# hello persistent world
```

---

## Exercise 4 — Data persists

```bash
kubectl exec pod-with-pvc -- cat /data/message.txt
# hello persistent world    ← still there after pod restart!
```

The data survived because it's stored in the PV (hostPath `/tmp/pv-pvc-lab`), not in the container layer.

---

## Exercise 5 — Pending PVC

```bash
kubectl get pvc pvc-pending
```
```
NAME          STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-pending   Pending                                      manual         10s
```

Events:
```
Warning  FailedBinding  no persistent volumes available for this claim and no storage class is set
```

---

## Challenge Answers

**1. Request more storage than any PV?**
PVC stays in `Pending` state indefinitely. Kubernetes keeps trying every few seconds to find a match. A `FailedBinding` event is recorded. To fix: either create a larger PV or reduce the PVC storage request.

**2. Two pods using the same PVC?**
It depends on the access mode. With `ReadWriteOnce` (RWO), the PVC can only be mounted on **one node** at a time — but multiple pods on the same node can use it. With `ReadWriteMany` (RWX), multiple pods across multiple nodes can mount it simultaneously.

**3. selector field in PVC?**
Allows you to target specific PVs by their labels. Only PVs with matching labels will be considered for binding. Example:
```yaml
selector:
  matchLabels:
    environment: production
    tier: ssd
```
Only PVs labelled `environment=production` and `tier=ssd` will be eligible.

**4. How Kubernetes chooses between multiple matching PVs?**
It uses a "best fit" algorithm — selects the **smallest PV whose capacity is >= the requested amount**. This minimises wasted space. If two PVs have the same size, selection is non-deterministic.

**5. Lost PVC phase?**
A PVC becomes `Lost` when its bound PV is **deleted while the PVC still exists**. The PVC can no longer function — any pod using it will fail to mount the volume. Data is likely gone. To recover: delete the PVC, create a new PV with the original data (if any was preserved), and create a new PVC.
