# 56 — Storage Classes

## What is a StorageClass?

A **StorageClass** is a template that tells Kubernetes **how to dynamically provision PersistentVolumes**. Instead of an admin manually creating PVs, a StorageClass defines the provisioner, parameters, and policies — and PVs are created automatically when a PVC is submitted.

---

## Static vs Dynamic Provisioning

```
Static:                               Dynamic:
Admin creates PV manually              PVC submitted
       ↓                                     ↓
PVC binds to existing PV              StorageClass provisions PV automatically
                                             ↓
                                      PV created and bound to PVC
```

---

## StorageClass Spec

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"  # make this the default
provisioner: kubernetes.io/aws-ebs          # who creates the PV
parameters:
  type: gp3                                  # provisioner-specific config
  iopsPerGB: "10"
  encrypted: "true"
reclaimPolicy: Delete                        # what happens to PV when PVC deleted
allowVolumeExpansion: true                   # can PVCs be resized?
volumeBindingMode: WaitForFirstConsumer      # when to bind
mountOptions:
- debug
```

---

## Common Provisioners

| Cloud/Platform | Provisioner |
|----------------|-------------|
| AWS EBS | `kubernetes.io/aws-ebs` or `ebs.csi.aws.com` |
| GCP Persistent Disk | `kubernetes.io/gce-pd` or `pd.csi.storage.gke.io` |
| Azure Disk | `kubernetes.io/azure-disk` or `disk.csi.azure.com` |
| Azure File | `kubernetes.io/azure-file` |
| NFS | `nfs.csi.k8s.io` or `k8s-sigs.io/nfs-subdir-external-provisioner` |
| Local | `kubernetes.io/no-provisioner` (manual binding) |
| minikube | `k8s.io/minikube-hostpath` |
| Rancher | `rancher.io/local-path` |

---

## Volume Binding Modes

| Mode | Behaviour |
|------|-----------|
| `Immediate` | PV is provisioned as soon as PVC is created (default) |
| `WaitForFirstConsumer` | PV is provisioned only when a pod using the PVC is scheduled — ensures the PV is created in the same availability zone as the pod |

> **Always use `WaitForFirstConsumer` on cloud providers** to avoid AZ mismatch (pod in us-east-1a but PV in us-east-1b).

---

## Default StorageClass

A cluster can have one default StorageClass. PVCs that don't specify a `storageClassName` use it automatically.

```bash
# Check current default
kubectl get storageclass
# The default has (default) next to its name

# Set a StorageClass as default
kubectl patch storageclass standard \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# Remove default from another class
kubectl patch storageclass old-default \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

---

## GKE Storage Classes Example

```yaml
# Standard HDD (default)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-standard
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer

# Premium SSD
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: premium-rwo
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

---

## Local StorageClass (no-provisioner)

For local SSDs or NVMe drives — requires `WaitForFirstConsumer`:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

PVs must still be created manually, but binding is deferred until scheduling.

---

## Useful Commands

```bash
# List StorageClasses
kubectl get storageclass
kubectl get sc                    # shorthand

# Describe a StorageClass
kubectl describe sc standard

# See which SC a PVC used
kubectl get pvc my-pvc -o jsonpath='{.spec.storageClassName}'

# See all PVs created by a SC
kubectl get pv -o custom-columns='NAME:.metadata.name,SC:.spec.storageClassName'
```

---

## Key Points for CKA

- A PVC with `storageClassName: ""` (empty string) explicitly opts out of dynamic provisioning
- A PVC with NO `storageClassName` field uses the default StorageClass
- `allowVolumeExpansion: true` is needed to resize PVCs
- `WaitForFirstConsumer` prevents topology/AZ mismatches on multi-zone clusters
