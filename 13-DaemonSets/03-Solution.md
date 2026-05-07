# DaemonSets — Solutions

## Section A — Existing DSes
On most clusters you'll see at least:
- `kube-proxy` (in `kube-system`)
- A CNI agent (calico-node, cilium, kindnet, weave-net...)
- Maybe `metrics-server`, `coredns`-related, log shippers

All have `tolerations: - operator: Exists` to land on every node.

## Section B — Create
On a single-node cluster, you get exactly 1 pod. On a 3-node cluster, exactly 3.

## Section C — nodeSelector
The DS controller respects `nodeSelector`. Pods on non-matching nodes are removed; matching nodes get pods. Removing the label removes the pod.

## Section D — Rolling Update
- Pods are replaced one at a time (because `maxUnavailable: 1`).
- `rollout history` shows revisions just like Deployments.
- `rollout restart` works the same way (annotation-based forced rollout).

## Section E — OnDelete
The controller does NOT auto-replace pods after a template change. You manually delete pods to pick up the new template. Useful for very sensitive cluster components where you want explicit, per-node control.

## Cheat Sheet
| Action | Command |
|---|---|
| Create | `kubectl apply -f ds.yaml` |
| List | `kubectl get ds -A` |
| Status | `kubectl rollout status ds/X` |
| Restart all pods | `kubectl rollout restart ds/X` |
| History | `kubectl rollout history ds/X` |
| Restrict by label | `spec.template.spec.nodeSelector` |
| Tolerate everything | `tolerations: [{operator: Exists}]` |
