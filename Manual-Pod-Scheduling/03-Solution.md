# Manual Pod Scheduling — Solutions

---

## Section A — Pin a Pod
The pod runs immediately. `kubectl describe` events show only kubelet actions (`Pulling`, `Pulled`, `Created`, `Started`) — no `Scheduled` event from `default-scheduler`. **Lesson:** the API server accepted the assignment as-is; the scheduler never looked at the pod.

## Section B — Non-existent Node
**Answer:** the pod is created in etcd but stays `Pending` forever because no kubelet for `does-not-exist` ever picks it up.
```
$ kubectl describe pod ghost | tail
... no Events
```
There is **no helpful error**. The scheduler would have given a clear `FailedScheduling`. Manual scheduling fails silently — you have to notice nothing is happening.

## Section C — Bypass Taint
The forced pod runs on the tainted node. Taints are evaluated by the scheduler. When you skip the scheduler, the taint has no effect.
```
$ kubectl get pod forced-pod -o wide
NAME          STATUS    NODE
forced-pod    Running   minikube
```
**Caveat:** `NoExecute` taints (which evict already-running pods) DO still apply at runtime, because they are evaluated by the kubelet, not the scheduler.

## Section D — Patch onto Pending Pod
**Answer:** the pod is created and may even reach `Running` briefly, but the kubelet may evict it because the request (500Gi) exceeds available memory. Or it gets `OOMKilled` if it actually tries to use the memory. The scheduler's fit check would have prevented this; manual scheduling does not.

## Section E — Immutable on Running Pod
```
The Pod "stay" is invalid: spec: Forbidden: pod updates may not change fields other than ...
```
**Lesson:** `nodeName` is in the immutable set. To "move" a pod, delete and recreate it.

## Section F — Multi-Node Comparison
Both pods run, on different nodes. The events tell the difference:
```
$ kubectl describe pod by-scheduler | grep Scheduled
  Normal  Scheduled    default-scheduler   Successfully assigned ...

$ kubectl describe pod by-hand | grep Scheduled
(empty)
```

## Section G — Scheduler Logs
Sample scheduler log line for the normal pod:
```
I0425 ... scheduler.go:548 "Successfully bound pod" pod="mansched/normal" node="worker-2"
```
No corresponding line appears for the manual pod.

---

## Bottom Line

Manual scheduling = "I take full responsibility for placement decisions." For 99% of workloads, prefer `nodeSelector`, `nodeAffinity`, or taints + tolerations. Reach for `nodeName` only in bootstrap, recovery, or custom-controller scenarios.
