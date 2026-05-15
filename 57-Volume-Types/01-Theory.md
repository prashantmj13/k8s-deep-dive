# 57 — Volume Types

## Overview

Kubernetes supports many volume types. Each serves a different purpose — from ephemeral scratch space to durable cloud disks.

---

## Volume Type Categories

| Category | Types | Persistence |
|----------|-------|-------------|
| Ephemeral | `emptyDir`, `configMap`, `secret`, `projected` | Lost when pod dies |
| Node-local | `hostPath`, `local` | Tied to the node |
| Network | `nfs`, `cephfs`, `glusterfs` | Durable, shared |
| Cloud | `awsElasticBlockStore`, `gcePersistentDisk`, `azureDisk` | Durable, cloud-managed |
| CSI | any `csi` driver | Depends on driver |

---

## 1. emptyDir

Created when a pod is assigned to a node. **Empty at start, deleted when pod is removed.** All containers in the pod share it.

```yaml
volumes:
- name: cache
  emptyDir: {}

# With memory-backed (faster, uses RAM)
- name: cache-mem
  emptyDir:
    medium: Memory
    sizeLimit: 512Mi
```

**Use cases:** Scratch space, inter-container communication, caching.

---

## 2. hostPath

Mounts a file or directory from the **host node's filesystem** into the pod.

```yaml
volumes:
- name: host-logs
  hostPath:
    path: /var/log
    type: Directory     # must exist

# type options:
# DirectoryOrCreate — create if not exists
# FileOrCreate      — create file if not exists
# Socket            — must be a Unix socket
# CharDevice        — must be a character device
# BlockDevice       — must be a block device
```

⚠️ **Security warning:** hostPath gives the container access to the host filesystem. Avoid in production unless necessary (e.g. DaemonSets that need host access like log collectors).

---

## 3. configMap

Mount a ConfigMap as a volume — each key becomes a file:

```yaml
volumes:
- name: config
  configMap:
    name: my-config
    items:                    # optional: select specific keys
    - key: app.properties
      path: app.properties   # filename in the mount

volumeMounts:
- name: config
  mountPath: /etc/config
  readOnly: true
```

Files update automatically when the ConfigMap changes (with ~60s delay).

---

## 4. secret

Mount a Secret as files (base64-decoded automatically):

```yaml
volumes:
- name: creds
  secret:
    secretName: my-secret
    defaultMode: 0400         # file permissions (octal)

volumeMounts:
- name: creds
  mountPath: /etc/creds
  readOnly: true
```

---

## 5. projected

Combines multiple volume sources into one mount point:

```yaml
volumes:
- name: all-in-one
  projected:
    sources:
    - secret:
        name: my-secret
    - configMap:
        name: my-config
    - serviceAccountToken:
        path: token
        expirationSeconds: 3600
```

---

## 6. NFS

Mount a Network File System share — supports ReadWriteMany:

```yaml
volumes:
- name: nfs-vol
  nfs:
    server: 192.168.1.100    # NFS server IP
    path: /exports/data      # exported path
    readOnly: false
```

---

## 7. PersistentVolumeClaim

Reference a PVC (most common for durable storage):

```yaml
volumes:
- name: data
  persistentVolumeClaim:
    claimName: my-pvc
    readOnly: false
```

---

## 8. CSI (Container Storage Interface)

Modern standard for storage drivers. Replaces in-tree drivers:

```yaml
volumes:
- name: csi-vol
  csi:
    driver: pd.csi.storage.gke.io      # CSI driver name
    volumeHandle: projects/myproject/zones/us-central1-a/disks/my-disk
    fsType: ext4
    readOnly: false
```

---

## 9. local

A local volume represents a mounted local storage device. Unlike `hostPath`, local volumes are aware of node topology:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage: 100Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /mnt/disks/ssd1
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - node01
```

---

## Volume Type Comparison

| Type | Shared across pods? | Survives pod restart? | Survives node restart? |
|------|--------------------|-----------------------|----------------------|
| emptyDir | Same pod only | ❌ | ❌ |
| hostPath | Same node | ✅ | ✅ |
| configMap/secret | Any pod | ✅ | ✅ |
| NFS | Any pod any node | ✅ | ✅ |
| PVC (cloud disk) | Depends on mode | ✅ | ✅ |
| CSI | Depends on driver | ✅ | ✅ |

---

## Deprecated In-tree Drivers (replaced by CSI)

Old in-tree drivers are deprecated in favour of CSI equivalents:

| Deprecated | CSI Replacement |
|-----------|----------------|
| `awsElasticBlockStore` | `ebs.csi.aws.com` |
| `gcePersistentDisk` | `pd.csi.storage.gke.io` |
| `azureDisk` | `disk.csi.azure.com` |
| `azureFile` | `file.csi.azure.com` |
