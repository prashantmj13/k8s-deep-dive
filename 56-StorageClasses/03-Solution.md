# 56 — Storage Classes | Solutions

---

## Exercise 4 — Dynamic PVC

```bash
kubectl get pvc dynamic-pvc
```
```
NAME          STATUS   VOLUME                                     CAPACITY   STORAGECLASS   AGE
dynamic-pvc   Bound    pvc-abc12345-...                           500Mi      standard       5s
```

```bash
kubectl get pv
```
```
NAME                                       CAPACITY   STORAGECLASS   STATUS   CLAIM
pvc-abc12345-...                           500Mi      standard       Bound    default/dynamic-pvc
```

A PV was automatically created by the `standard` StorageClass provisioner — no admin action needed.

---

## Exercise 7 — Default SC change

```bash
kubectl get sc
```
```
NAME                 PROVISIONER                RECLAIMPOLICY   VOLUMEBINDINGMODE
local-manual(default) kubernetes.io/no-provisioner  Retain       WaitForFirstConsumer
standard             k8s.io/minikube-hostpath   Delete          Immediate
```

---

## Challenge Answers

**1. Immediate vs WaitForFirstConsumer?**
- `Immediate`: The PV is provisioned as soon as the PVC is created — before any pod requests it. On multi-zone clusters this can cause problems: the PV may be created in a different availability zone than the pod that eventually uses it, causing mount failures.
- `WaitForFirstConsumer`: PV provisioning is deferred until a pod using the PVC is actually scheduled. Kubernetes knows the pod's node/zone and provisions the PV in the right location. Always recommended on cloud providers.

**2. PVC with no storageClassName?**
It uses the **cluster's default StorageClass** (the one annotated with `is-default-class: true`). If no default SC exists, the PVC stays `Pending`. To explicitly opt out of all StorageClasses and use only static PVs, set `storageClassName: ""` (empty string, not omitted).

**3. allowVolumeExpansion: true?**
Allows PVCs using this StorageClass to be resized after creation by patching the PVC's `spec.resources.requests.storage` to a larger value. The underlying cloud disk (EBS, GCP PD, etc.) is expanded. Not all backends support shrinking — you can only increase the size.

**4. Change StorageClass after creation?**
StorageClasses are mostly **immutable** after creation. You cannot change the `provisioner`, `parameters`, or `volumeBindingMode`. You can change the `reclaimPolicy` and annotations. If you need different parameters, delete and recreate the StorageClass (existing PVs are unaffected).

**5. kubernetes.io/no-provisioner?**
This is the "manual provisioner" — it tells Kubernetes that PVs must be created manually by an admin. No automatic provisioning happens. It's used when you want to use a StorageClass to control binding mode (especially `WaitForFirstConsumer`) for local storage, but still manage PVs yourself.
