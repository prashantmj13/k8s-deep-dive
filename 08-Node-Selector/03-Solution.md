# nodeSelector — Solutions

---

## Section C — Match Failure
```
Events:
  Warning  FailedScheduling   default-scheduler   0/1 nodes are available:
                              1 node(s) didn't match Pod's node affinity/selector.
```
The scheduler doesn't tell you *which* labels mismatched — it only says "didn't match." For more detail, you'd inspect both the pod's `spec.nodeSelector` and the node's `metadata.labels`.

## Section D — Mid-Flight Label Change
**Answer:** the pod keeps running. `nodeSelector` is enforced **only at scheduling time**. Once the scheduler binds the pod to a node, the labels could vanish from the node and the pod would still happily run there. To re-evaluate, you'd need to delete and recreate the pod.

This is one of the trade-offs of `nodeSelector` (and `requiredDuringSchedulingIgnoredDuringExecution` in nodeAffinity — note the "IgnoredDuringExecution" part). Kubernetes does NOT have a `requiredDuringExecution` mode that would reschedule the pod on label changes.

## Section E — No OR
The only way to express OR with `nodeSelector` is to label all valid nodes with the same `flavor: X` value. The matching is still equality. For true `In` / `NotIn` / `Exists` matching, switch to `nodeAffinity.matchExpressions`.

## Section F — Dedicated Nodes
The pattern is "taint to keep others off + label to attract own workload + toleration to allow ourselves in." Without the taint, other pods can land on your dedicated node. Without the label/selector, your pod might land elsewhere. Without the toleration, your pod can't even land on the tainted node.

## Section G — Portable Labels
`kubernetes.io/os` and `kubernetes.io/arch` are universal. `topology.kubernetes.io/zone` is set when the cloud provider integration is active. Use these instead of inventing custom keys when expressing the same concept — your manifests will work on EKS, GKE, AKS, and on-prem identically.

---

## Cheat Sheet

| Goal | Mechanism |
|---|---|
| Pod must run on node with key=value | `nodeSelector` (this folder) |
| Pod prefers nodes with key=value (soft) | `nodeAffinity` |
| Pod must avoid certain nodes | Taint the node + don't add toleration to pod |
| OR / set-based matching | `nodeAffinity.matchExpressions` |
| Don't want others on this node | Taint the node |

| Command | Use |
|---|---|
| `kubectl label nodes N k=v` | Add label |
| `kubectl label nodes N k=v --overwrite` | Update |
| `kubectl label nodes N k-` | Remove |
| `kubectl get nodes -L k1,k2` | Show labels as columns |
| `kubectl get nodes -l k=v` | Filter nodes |
