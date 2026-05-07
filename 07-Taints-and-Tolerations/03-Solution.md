# Taints and Tolerations — Solutions

---

## Section B — NoSchedule
```
$ kubectl describe pod regular | tail
Events:
  Warning  FailedScheduling   default-scheduler  0/1 nodes are available: 1 node(s) had untolerated taint {gpu: true}.
```
**Lesson:** the message tells you exactly which taint blocked the pod.

## Section C — Existing Pods Untouched
The pod keeps running. **NoSchedule** only affects placement of new pods, not running ones.

## Section D — NoExecute Eviction
```
$ kubectl get pods -w
plain    1/1   Running       0   2m
plain    1/1   Terminating   0   2m   <- evicted by NoExecute
tough    1/1   Running       0   2m   <- survives
```

## Section E — tolerationSeconds
The pod stays running for exactly the seconds you specified, then is evicted by the kubelet. This is how you do graceful drains: workloads with `tolerationSeconds: 60` get a minute to finish in-flight work before SIGTERM.

## Section F — Catch-all Toleration
A pod with `operator: Exists` and no key tolerates EVERY taint, including `NoExecute`. This is needed for very privileged DaemonSets like `kube-proxy` and CNI agents that must run on every node regardless of taints.

## Section G — Dedicated Nodes Pattern
- Taint without selector → tolerating pod might run there but might not.
- Selector without taint → tolerating pod runs there, but other pods can too.
- Taint + selector + toleration → only this workload runs there. **The right pattern.**

## Section H — DaemonSet Tolerations
Sample from `kube-proxy`:
```yaml
tolerations:
- operator: Exists                        # tolerate everything
- key: CriticalAddonsOnly
  operator: Exists
- effect: NoExecute
  operator: Exists
```
This is why DaemonSet pods reach every node, even tainted control-plane and pressure-state nodes.

## Section I — Drain
`kubectl drain` does three things:
1. Cordons the node (adds `node.kubernetes.io/unschedulable:NoSchedule`).
2. Evicts pods (calls the eviction API for each — respects PodDisruptionBudgets).
3. Skips DaemonSet pods (per `--ignore-daemonsets`).

`kubectl uncordon` removes the taint. The node is schedulable again.

---

## Cheat Sheet

| Action | Command |
|---|---|
| Add NoSchedule taint | `kubectl taint nodes N k=v:NoSchedule` |
| Add NoExecute taint | `kubectl taint nodes N k=v:NoExecute` |
| Remove all taints with key | `kubectl taint nodes N k-` |
| Remove specific effect | `kubectl taint nodes N k:NoSchedule-` |
| View taints | `kubectl describe node N \| grep -A2 Taints` |
| Drain | `kubectl drain N --ignore-daemonsets --delete-emptydir-data` |
| Cordon (add unschedulable taint) | `kubectl cordon N` |
| Uncordon | `kubectl uncordon N` |

| Pattern | Used for |
|---|---|
| Taint + tolerate + nodeSelector | Dedicated nodes (GPU, ML, prod) |
| `operator: Exists` (no key) | "Tolerate everything" — DaemonSets |
| `tolerationSeconds: 30` | Custom faster eviction on node failure |
| `PreferNoSchedule` | Soft preference; not a hard block |
