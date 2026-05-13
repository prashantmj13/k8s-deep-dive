# 41 — Persistent Key/Value Store (etcd)

## What is etcd?

etcd is a **distributed, consistent key-value store** that serves as Kubernetes' backing database. Every object you create (pods, deployments, secrets, configmaps, RBAC rules) is stored as a key-value pair in etcd.

---

## Key Properties

| Property | Description |
|----------|-------------|
| **Consistent** | All reads return the most recent write (strong consistency) |
| **Distributed** | Runs as a cluster for high availability |
| **Watch** | Clients can subscribe to key changes in real time |
| **Reliable** | Uses the Raft consensus algorithm |
| **Encrypted transport** | All communication over TLS |

---

## How Kubernetes Uses etcd

```
kubectl create deployment nginx ...
           ↓
    kube-apiserver
           ↓
    writes to etcd: /registry/deployments/default/nginx
           ↓
controller-manager watches etcd → creates ReplicaSet
           ↓
scheduler watches etcd → assigns pods to nodes
           ↓
kubelet watches etcd (via API server) → starts containers
```

The API server is the **only component** that talks directly to etcd. All other components go through the API server.

---

## etcd Data Layout in Kubernetes

All Kubernetes objects are stored under `/registry/`:

```
/registry/
├── namespaces/
│   ├── default
│   └── kube-system
├── pods/
│   └── default/
│       └── nginx-xxx
├── deployments/
│   └── default/
│       └── nginx
├── secrets/
│   └── default/
│       └── my-secret
├── configmaps/
│   └── default/
│       └── my-config
├── serviceaccounts/
├── clusterroles/
├── clusterrolebindings/
└── ...
```

---

## Querying etcd Directly

```bash
export ETCDCTL_API=3
export ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt
export ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt
export ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key
export ETCDCTL_ENDPOINTS=https://127.0.0.1:2379

# List all keys
etcdctl get / --prefix --keys-only

# Get a specific key (pod)
etcdctl get /registry/pods/default/nginx-xxx

# Count total keys
etcdctl get / --prefix --keys-only | wc -l

# Watch for changes to pods
etcdctl watch /registry/pods --prefix
```

---

## etcd Cluster Topology

### Stacked etcd (default in kubeadm)
etcd runs on the same nodes as the control plane. Simple but less resilient.

### External etcd
etcd runs on dedicated nodes separate from the control plane. More resilient, recommended for production.

```
Stacked:                    External:
┌─────────────────┐         ┌──────────┐   ┌──────────┐
│  Control Plane  │         │ Control  │   │  etcd    │
│  ┌───────────┐  │         │  Plane   │──▶│  Cluster │
│  │   etcd    │  │         └──────────┘   └──────────┘
│  └───────────┘  │
└─────────────────┘
```

---

## etcd HA: Quorum

etcd uses the **Raft consensus algorithm**. Writes are committed only when a majority of members (quorum) acknowledge them.

| Cluster Size | Quorum (majority) | Fault Tolerance |
|-------------|-------------------|-----------------|
| 1 | 1 | 0 nodes can fail |
| 3 | 2 | 1 node can fail |
| 5 | 3 | 2 nodes can fail |
| 7 | 4 | 3 nodes can fail |

> Always use **odd numbers** for etcd clusters (3 or 5 for production).

---

## etcd Performance Considerations

- etcd is sensitive to disk I/O — use SSDs
- Default data size limit: 8GB (configurable with `--quota-backend-bytes`)
- Run `etcdctl defrag` periodically to reclaim space
- Monitor with `etcdctl endpoint status`

---

## Checking etcd Health

```bash
etcdctl endpoint health
etcdctl endpoint status --write-out=table
etcdctl alarm list   # check for any alarms (e.g. NOSPACE)
```
