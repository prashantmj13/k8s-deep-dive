# 33 — ETCDCTL and ETCDUTL | Solutions

---

## Exercise 3 — Endpoint health

```
https://127.0.0.1:2379 is healthy: successfully committed proposal: took = 2.3ms
```

Endpoint status:
```
+---------------------------+------------------+---------+---------+-----------+
|         ENDPOINT          |        ID        | VERSION | DB SIZE | IS LEADER |
+---------------------------+------------------+---------+---------+-----------+
| https://127.0.0.1:2379    | 8e9e05c52164694d |  3.5.9  |  5.2 MB |      true |
+---------------------------+------------------+---------+---------+-----------+
```

---

## Exercise 4 — Member list

```
+------------------+---------+--------+---------------------------+---------------------------+
|        ID        | STATUS  |  NAME  |        PEER ADDRS         |       CLIENT ADDRS        |
+------------------+---------+--------+---------------------------+---------------------------+
| 8e9e05c52164694d | started | node1  | https://127.0.0.1:2380    | https://127.0.0.1:2379    |
+------------------+---------+--------+---------------------------+---------------------------+
```

---

## Exercise 5 — Registry keys

```bash
etcdctl get /registry --prefix --keys-only | head -20
```
```
/registry/apiextensions.k8s.io/customresourcedefinitions/...
/registry/clusterrolebindings/...
/registry/clusterroles/...
/registry/configmaps/default/...
/registry/deployments/default/...
/registry/namespaces/default
/registry/pods/default/...
/registry/secrets/default/...
/registry/serviceaccounts/default/...
/registry/services/...
```

---

## Exercise 6 — Snapshot

```
+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| 4a3bc918 |    45231 |       1132 |     5.2 MB |
+----------+----------+------------+------------+
```

---

## Exercise 8 — Restore directory

```bash
ls /tmp/etcd-test-restore/member/
# Output: snap  wal
```

The restore created `member/snap` and `member/wal` directories — the standard etcd data layout.

---

## Challenge Answers

**1. Required env variable?**
`ETCDCTL_API=3`. Without it, etcdctl defaults to API v2 which is deprecated and won't work correctly with Kubernetes etcd clusters.

**2. Find etcd endpoint on kubeadm?**
```bash
kubectl describe pod etcd-controlplane -n kube-system | grep advertise-client-urls
```
Usually `https://127.0.0.1:2379` or `https://<control-plane-IP>:2379`.

**3. Where are secrets in etcd?**
Under `/registry/secrets/<namespace>/<name>`. They're stored as base64-encoded values (not encrypted unless Encryption at Rest is configured).

**4. etcdctl vs etcdutl snapshot restore?**
`etcdctl snapshot restore` (older method) — connects to a running etcd instance. `etcdutl snapshot restore` (recommended since etcd 3.5) — fully offline, reads the snapshot file directly. More reliable for disaster recovery because it doesn't need a running cluster.

**5. Check etcd health without restarting?**
```bash
etcdctl endpoint health
etcdctl endpoint status --write-out=table
```
These are read-only checks against the live cluster — safe to run anytime.
