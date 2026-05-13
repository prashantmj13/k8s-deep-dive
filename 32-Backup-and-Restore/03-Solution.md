# 32 — Backup and Restore | Solutions & Expected Outputs

---

## Exercise 3 — Snapshot taken

```
/backup/etcd-snapshot.db  3.2M
```

```bash
# Confirm
{"level":"info","ts":"...","msg":"snapshot saved","path":"/backup/etcd-snapshot.db"}
```

---

## Exercise 4 — Snapshot status

```
+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| a8f3c91e |     9821 |        892 |     3.2 MB |
+----------+----------+------------+------------+
```

---

## Exercise 5 — After deleting namespace

```bash
kubectl get namespace backup-test
Error from server (NotFound): namespaces "backup-test" not found
```

---

## Exercise 6 — Restore output

```bash
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restored
```
```
{"level":"info","msg":"restoring snapshot","path":"/backup/etcd-snapshot.db","wal-dir":"/var/lib/etcd-restored/member/wal"}
{"level":"info","msg":"restored snapshot","path":"/backup/etcd-snapshot.db"}
```

After cluster comes back:
```bash
kubectl get nodes
NAME           STATUS   VERSION
controlplane   Ready    v1.29.0
```

---

## Exercise 7 — Restored resources

```bash
kubectl get namespace backup-test
NAME          STATUS   AGE
backup-test   Active   1s

kubectl get all -n backup-test
NAME                          READY   STATUS    RESTARTS   AGE
pod/webapp-xxx-aaa            1/1     Running   0          1m
pod/webapp-xxx-bbb            1/1     Running   0          1m
pod/webapp-xxx-ccc            1/1     Running   0          1m

NAME                     READY   UP-TO-DATE   AVAILABLE
deployment.apps/webapp   3/3     3            3

kubectl get configmap app-config -n backup-test
NAME         DATA   AGE
app-config   1      1m
```

All resources restored exactly as they were before deletion.

---

## Challenge Answers

**1. save vs restore?**
- `snapshot save`: connects to a **running** etcd cluster and captures a point-in-time snapshot of all data into a file.
- `snapshot restore`: reads a snapshot file and writes a new etcd data directory from it. etcd must NOT be running when you restore (it's an offline operation).

**2. Why stop API server?**
The API server connects to etcd. If the API server is running during a restore, it will continue reading from the old etcd data. Also, stopping it prevents new writes that would be lost after restore. On kubeadm clusters, moving the static pod manifest out of `/etc/kubernetes/manifests/` is the clean way to stop it.

**3. etcd certificate locations (kubeadm)?**
```
/etc/kubernetes/pki/etcd/ca.crt    ← CA cert
/etc/kubernetes/pki/etcd/server.crt ← Server cert
/etc/kubernetes/pki/etcd/server.key ← Server key
```
These are required for `etcdctl` to connect to etcd securely.

**4. snapshot status?**
Shows: hash (integrity check), revision (etcd internal version), total keys (number of objects stored), total size (file size). Use it to verify the snapshot is not corrupted and to understand how much data is captured.

**5. etcdctl vs etcdutl?**
- `etcdctl`: requires a running etcd server. Used for `snapshot save`, `get`, `put`, `del`, `watch`.
- `etcdutl`: offline tool, no running etcd needed. Used for `snapshot restore`, `snapshot status`, `defrag` on data files directly. In etcd 3.5+, `etcdutl` is the recommended way to restore snapshots.
