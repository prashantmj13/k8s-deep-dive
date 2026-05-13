# 41 — Persistent Key/Value Store | Exercises

> **Cluster:** kubeadm cluster
> **Estimated time:** 15 minutes

---

## Exercise 1 — Connect and check health

```bash
export ETCDCTL_API=3
export ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt
export ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt
export ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key
export ETCDCTL_ENDPOINTS=https://127.0.0.1:2379

etcdctl endpoint health
etcdctl endpoint status --write-out=table
```

---

## Exercise 2 — Explore the key space

```bash
# List all top-level registry keys
etcdctl get /registry --prefix --keys-only | \
  awk -F/ '{print "/"$2"/"$3}' | sort -u

# Count total objects stored
etcdctl get /registry --prefix --keys-only | wc -l
```

---

## Exercise 3 — Find a specific pod in etcd

```bash
# Create a test pod first
kubectl run etcd-test --image=nginx

# Find its key in etcd
etcdctl get /registry/pods/default --prefix --keys-only | grep etcd-test

# Read its value (will be binary/protobuf, look for readable strings)
etcdctl get /registry/pods/default/etcd-test | strings | grep -E "image:|namespace:|name:"
```

---

## Exercise 4 — Watch for changes in real time

Open two terminals.

Terminal 1 (watch):
```bash
etcdctl watch /registry/pods/default --prefix
```

Terminal 2 (create pod):
```bash
kubectl run watch-test --image=nginx
kubectl delete pod watch-test
```

> **Observe:** Terminal 1 shows PUT and DELETE events as they happen.

---

## Exercise 5 — Check quorum

```bash
etcdctl member list --write-out=table
```

Note: In a single-node kubeadm cluster there is only one member. For HA you'd see 3 or 5.

---

## Exercise 6 — Alarm check and defrag

```bash
etcdctl alarm list

# Defrag to reclaim space (safe to run on live cluster, brief latency spike)
etcdctl defrag
etcdctl endpoint status --write-out=table
```

---

## Exercise 7 — Cleanup

```bash
kubectl delete pod etcd-test 2>/dev/null; true
```

---

## Challenge Questions

1. What is the Raft consensus algorithm used for in etcd?
2. Why should etcd cluster size always be an odd number?
3. Which Kubernetes component is the only one allowed to talk directly to etcd?
4. What does `etcdctl watch` do and how does Kubernetes use this mechanism?
5. What happens when etcd runs out of disk space?
