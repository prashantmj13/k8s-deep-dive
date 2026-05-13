# 32 — Backup and Restore | Exercises

> **Cluster:** kubeadm-based cluster (control plane access required)
> **Estimated time:** 30 minutes
> **Run all etcdctl commands on the control plane node**

---

## Exercise 1 — Check etcd is running

```bash
kubectl get pods -n kube-system | grep etcd
kubectl describe pod etcd-controlplane -n kube-system | grep -E "Image:|Command:"
```

Find the etcd endpoint and certificate paths.

---

## Exercise 2 — Set up a test workload

```bash
kubectl create namespace backup-test
kubectl create deployment webapp --image=nginx --replicas=3 -n backup-test
kubectl create configmap app-config --from-literal=env=production -n backup-test
kubectl get all -n backup-test
```

This is the state we want to restore after simulating data loss.

---

## Exercise 3 — Take an etcd snapshot

```bash
mkdir -p /backup

ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

ls -lh /backup/etcd-snapshot.db
```

---

## Exercise 4 — Verify the snapshot

```bash
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot.db \
  --write-out=table
```

Note the TOTAL KEYS count.

---

## Exercise 5 — Simulate data loss

```bash
kubectl delete namespace backup-test
kubectl get namespace backup-test
```

The namespace and all its resources are now gone.

---

## Exercise 6 — Restore from snapshot

```bash
# Step 1: Stop API server
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/kube-apiserver.yaml

# Wait for API server to stop
sleep 10

# Step 2: Restore snapshot to new data directory
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restored

# Step 3: Update etcd manifest to use restored data
sed -i 's|/var/lib/etcd|/var/lib/etcd-restored|g' /etc/kubernetes/manifests/etcd.yaml

# Step 4: Restore API server
mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/kube-apiserver.yaml

# Step 5: Wait for cluster to come back
sleep 30
kubectl get nodes
```

---

## Exercise 7 — Verify restore

```bash
kubectl get namespace backup-test
kubectl get all -n backup-test
kubectl get configmap app-config -n backup-test
```

The namespace and all resources should be restored.

---

## Exercise 8 — Export manifests as alternative backup

```bash
kubectl get all --all-namespaces -o yaml > /backup/all-resources.yaml
wc -l /backup/all-resources.yaml
```

---

## Exercise 9 — Cleanup

```bash
kubectl delete namespace backup-test
```

---

## Challenge Questions

1. What is the difference between `etcdctl snapshot save` and `etcdctl snapshot restore`?
2. Why do you need to stop the API server before restoring etcd?
3. Where are the etcd certificates located in a kubeadm cluster?
4. What does `etcdctl snapshot status` tell you?
5. What is the difference between `etcdctl` and `etcdutl`?
