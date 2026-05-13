# 33 — Working with ETCDCTL and ETCDUTL

## Overview

| Tool | Purpose | Requires running etcd? |
|------|---------|----------------------|
| `etcdctl` | Interact with a live etcd cluster | Yes |
| `etcdutl` | Offline operations on etcd data files | No |

Both tools ship with etcd. Always set `ETCDCTL_API=3` — API v2 is deprecated.

---

## etcdctl — Common Commands

### Snapshot operations
```bash
# Save snapshot
ETCDCTL_API=3 etcdctl snapshot save /backup/snap.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Check snapshot status
ETCDCTL_API=3 etcdctl snapshot status /backup/snap.db --write-out=table
```

### Key-value operations
```bash
# Get a key
ETCDCTL_API=3 etcdctl get /registry/pods/default/mypod \
  --endpoints=... --cacert=... --cert=... --key=...

# List all keys with a prefix
ETCDCTL_API=3 etcdctl get /registry --prefix --keys-only \
  --endpoints=... --cacert=... --cert=... --key=...

# Put a key
ETCDCTL_API=3 etcdctl put mykey "myvalue" ...

# Delete a key
ETCDCTL_API=3 etcdctl del mykey ...

# Watch a key for changes
ETCDCTL_API=3 etcdctl watch /registry/pods --prefix ...
```

### Member operations
```bash
# List cluster members
ETCDCTL_API=3 etcdctl member list --write-out=table \
  --endpoints=... --cacert=... --cert=... --key=...

# Check endpoint health
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=... --cacert=... --cert=... --key=...

# Check endpoint status
ETCDCTL_API=3 etcdctl endpoint status --write-out=table \
  --endpoints=... --cacert=... --cert=... --key=...
```

---

## Simplify with Environment Variables

Instead of passing flags every time, export them:

```bash
export ETCDCTL_API=3
export ETCDCTL_ENDPOINTS=https://127.0.0.1:2379
export ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt
export ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt
export ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key

# Now commands are much shorter
etcdctl endpoint health
etcdctl snapshot save /backup/snap.db
etcdctl member list
```

---

## etcdutl — Offline Tool (etcd 3.5+)

`etcdutl` operates directly on etcd data files — no running etcd needed.

### Restore a snapshot (preferred in etcd 3.5+)
```bash
etcdutl snapshot restore /backup/snap.db \
  --data-dir=/var/lib/etcd-restored \
  --name=controlplane \
  --initial-cluster=controlplane=https://127.0.0.1:2380 \
  --initial-advertise-peer-urls=https://127.0.0.1:2380
```

### Check snapshot status offline
```bash
etcdutl snapshot status /backup/snap.db --write-out=table
```

### Defragment data directory
```bash
# Stop etcd first, then:
etcdutl defrag --data-dir=/var/lib/etcd
```

---

## Finding etcd Endpoint and Certs

On a kubeadm cluster, inspect the etcd pod:

```bash
kubectl describe pod etcd-controlplane -n kube-system | grep -E "\-\-"
```

Key flags to look for:
```
--advertise-client-urls=https://192.168.1.10:2379   ← endpoint
--cert-file=/etc/kubernetes/pki/etcd/server.crt
--key-file=/etc/kubernetes/pki/etcd/server.key
--trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
--data-dir=/var/lib/etcd
```

---

## etcd Data in Kubernetes

All Kubernetes objects are stored in etcd under `/registry/`:

| Path | Content |
|------|---------|
| `/registry/pods/default/` | Pods in default namespace |
| `/registry/deployments/` | Deployments |
| `/registry/secrets/` | Secrets (base64) |
| `/registry/namespaces/` | Namespaces |
| `/registry/serviceaccounts/` | Service accounts |

---

## Key Tips for CKA Exam

- Always export `ETCDCTL_API=3` before using etcdctl
- The `--endpoints`, `--cacert`, `--cert`, `--key` flags are always needed
- For restore, prefer `etcdutl` on newer clusters
- Remember to update `--data-dir` in etcd static pod manifest after restore
- Use `kubectl describe pod etcd-controlplane -n kube-system` to find all cert paths
