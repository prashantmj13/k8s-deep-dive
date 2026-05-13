# 33 — ETCDCTL and ETCDUTL | Exercises

> **Cluster:** kubeadm cluster (control plane access required)
> **Estimated time:** 20 minutes

---

## Exercise 1 — Find etcd details

```bash
kubectl get pod etcd-controlplane -n kube-system -o yaml | grep -E "image:|--"
```

Note down: `--endpoints`, `--cert-file`, `--key-file`, `--trusted-ca-file`, `--data-dir`

---

## Exercise 2 — Export environment variables

```bash
export ETCDCTL_API=3
export ETCDCTL_ENDPOINTS=https://127.0.0.1:2379
export ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt
export ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt
export ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key

echo $ETCDCTL_API
```

---

## Exercise 3 — Check endpoint health and status

```bash
etcdctl endpoint health
etcdctl endpoint status --write-out=table
```

---

## Exercise 4 — List cluster members

```bash
etcdctl member list --write-out=table
```

---

## Exercise 5 — Explore keys in etcd

```bash
# List all top-level registry keys
etcdctl get /registry --prefix --keys-only | head -20

# List all pods in default namespace
etcdctl get /registry/pods/default --prefix --keys-only

# Get a specific pod (replace pod-name)
kubectl get pods
etcdctl get /registry/pods/default/<pod-name> | strings | head -20
```

---

## Exercise 6 — Take a snapshot

```bash
mkdir -p /backup
etcdctl snapshot save /backup/etcd-snap.db
etcdctl snapshot status /backup/etcd-snap.db --write-out=table
ls -lh /backup/etcd-snap.db
```

---

## Exercise 7 — Use etcdutl for offline status check

```bash
which etcdutl || echo "etcdutl not found - check etcd version"

etcdutl snapshot status /backup/etcd-snap.db --write-out=table
```

---

## Exercise 8 — Restore using etcdutl

```bash
etcdutl snapshot restore /backup/etcd-snap.db \
  --data-dir=/tmp/etcd-test-restore

ls /tmp/etcd-test-restore/member/
```

This creates a new data directory — useful to verify restore works without affecting the live cluster.

---

## Challenge Questions

1. What environment variable must always be set when using etcdctl?
2. How do you find the etcd endpoint URL on a kubeadm cluster?
3. Where in etcd are Kubernetes secrets stored?
4. What is the difference between `etcdctl snapshot restore` and `etcdutl snapshot restore`?
5. How can you check if etcd is healthy without restarting anything?
