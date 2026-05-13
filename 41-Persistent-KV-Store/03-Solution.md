# 41 — Persistent Key/Value Store | Solutions

---

## Exercise 2 — Key space

```
/registry/apiextensions
/registry/apiregistration
/registry/clusterrolebindings
/registry/clusterroles
/registry/configmaps
/registry/deployments
/registry/events
/registry/namespaces
/registry/pods
/registry/replicasets
/registry/secrets
/registry/serviceaccounts
/registry/services
```

Total keys: typically 800–1500 on a fresh cluster.

---

## Exercise 4 — Watch output

```
PUT
/registry/pods/default/watch-test
<binary protobuf data>

DELETE
/registry/pods/default/watch-test
<binary data>
```

etcd sends events in real time — this is exactly how Kubernetes controllers work internally.

---

## Exercise 5 — Member list

```
+------------------+---------+--------+---------------------------+---------------------------+
|        ID        | STATUS  |  NAME  |        PEER ADDRS         |       CLIENT ADDRS        |
+------------------+---------+--------+---------------------------+---------------------------+
| 8e9e05c52164694d | started |  node1 | https://127.0.0.1:2380    | https://127.0.0.1:2379    |
+------------------+---------+--------+---------------------------+---------------------------+
```

Single member = no fault tolerance. In production you'd see 3 or 5 members.

---

## Challenge Answers

**1. Raft consensus algorithm?**
Raft ensures all etcd members agree on the order of writes. One member is elected **leader** and handles all writes. The leader replicates each write to followers. A write is committed only when a majority (quorum) of members acknowledge it. This guarantees consistency even if some nodes fail.

**2. Why odd numbers?**
Quorum = majority = (N/2)+1. With an even number like 4, quorum is 3 — same as with 3 nodes, but you're running an extra node with no additional fault tolerance. Odd numbers maximise fault tolerance per node added: 3 nodes (tolerate 1 failure), 5 nodes (tolerate 2 failures), 7 nodes (tolerate 3 failures).

**3. Only component talking to etcd?**
The **kube-apiserver**. All other components (controller-manager, scheduler, kubelet, kubectl) talk to the API server, which then reads/writes etcd. This simplifies security (only one component needs etcd certs) and lets the API server implement caching, watches, and validation before touching etcd.

**4. etcdctl watch / how Kubernetes uses it?**
`etcdctl watch` subscribes to changes on a key or prefix. Kubernetes controllers use the same watch mechanism via the API server — they register informers that are backed by etcd watches. When a pod is created, etcd sends a watch notification to the API server, which forwards it to the scheduler and controller manager. This is the core of Kubernetes' event-driven reconciliation loop.

**5. etcd out of disk space?**
etcd triggers a `NOSPACE` alarm and **stops accepting writes**. The cluster becomes read-only. This causes the API server to reject all mutating requests (create, update, delete). Fix: free disk space or increase quota with `--quota-backend-bytes`, then run `etcdctl alarm disarm`. Monitor disk with `etcdctl endpoint status`.
