# 30 — Cluster Maintenance: OS Upgrade | Solutions

---

## Exercise 2 — Cordon output

```
NAME           STATUS                     ROLES    AGE
controlplane   Ready                      master   10d
node01         Ready,SchedulingDisabled   worker   10d
```

After scaling to 6 replicas, new pods only land on non-cordoned nodes:
```
nginx-xxx-aaa   1/1   Running   controlplane
nginx-xxx-bbb   1/1   Running   controlplane
nginx-xxx-ccc   1/1   Running   controlplane   ← 3 new ones, none on node01
```

---

## Exercise 3 — Drain output

```bash
kubectl drain node01 --ignore-daemonsets --delete-emptydir-data
```
```
node/node01 cordoned
evicting pod maintenance-lab/nginx-xxx-ddd
evicting pod maintenance-lab/nginx-xxx-eee
pod/nginx-xxx-ddd evicted
pod/nginx-xxx-eee evicted
node/node01 drained
```

All pods now run on remaining nodes:
```
nginx-xxx-aaa   1/1   Running   controlplane
nginx-xxx-bbb   1/1   Running   controlplane
nginx-xxx-fff   1/1   Running   controlplane   ← rescheduled from node01
```

---

## Exercise 5 — Uncordon

```
NAME           STATUS   ROLES    AGE
controlplane   Ready    master   10d
node01         Ready    worker   10d    ← SchedulingDisabled gone
```

Existing pods stay where they are. Only **new** pods will be eligible to schedule on node01.

---

## Exercise 6 — Bare pod drain failure

```
error: cannot delete Pods declare no controller (use --force to override):
maintenance-lab/bare-pod
```

After `--force`:
```bash
kubectl get pods
# bare-pod is gone — it was evicted and NOT rescheduled
```

---

## Challenge Answers

**1. Cordon vs Drain?**
- `cordon`: marks node unschedulable, existing pods stay running
- `drain`: cordons the node AND evicts all pods from it gracefully

**2. Node back within 5 minutes?**
Kubernetes does NOT reschedule pods if the node returns within `pod-eviction-timeout` (default 5 min). Pods are still shown as running on the node. Once the node is back and passes health checks, they continue normally.

**3. DaemonSet pods ignored?**
DaemonSet pods are managed by the DaemonSet controller which immediately recreates them on the node. They can't be "evicted" to another node by definition — one per node is their purpose. So drain skips them with `--ignore-daemonsets`.

**4. PodDisruptionBudget?**
A PDB defines minimum available (or maximum unavailable) replicas for a workload. `kubectl drain` respects PDBs — if evicting a pod would violate the budget, drain pauses and waits. This ensures high availability during maintenance. Override with `--disable-eviction`.

**5. Pods move back after uncordon?**
No. Kubernetes does NOT rebalance pods. Once rescheduled to other nodes, they stay there. Only new pod creations or pod restarts will consider the uncordoned node. Use `kubectl rollout restart deployment` if you want pods redistributed.
