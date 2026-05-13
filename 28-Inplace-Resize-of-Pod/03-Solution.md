# 28 — In-place Resize of Pod | Solutions & Expected Outputs

---

## Exercise 3 — Current allocated resources

```json
{
  "limits": { "cpu": "200m", "memory": "128Mi" },
  "requests": { "cpu": "100m", "memory": "64Mi" }
}
```

`allocatedResources` matches `requests` initially.

---

## Exercise 4 — CPU resize (no restart)

```bash
kubectl get pod pod-resize -o jsonpath='{.status.resize}'
# Output: InProgress  (then disappears once complete)

kubectl get pod pod-resize -o jsonpath='{.status.containerStatuses[0].allocatedResources}'
# Output: {"cpu":"200m","memory":"64Mi"}   ← CPU request updated

kubectl get pod pod-resize
```
```
NAME         READY   STATUS    RESTARTS   AGE
pod-resize   1/1     Running   0          3m
```
**RESTARTS stays at 0** — CPU resize is hot-patched with `NotRequired`.

---

## Exercise 5 — Memory resize (restart)

```bash
kubectl get pod pod-resize -w
```
```
NAME         READY   STATUS    RESTARTS   AGE
pod-resize   1/1     Running   0          5m
pod-resize   1/1     Running   1          5m10s
```

**RESTARTS increments by 1** because memory resizePolicy is `RestartContainer`.

After restart:
```bash
kubectl get pod pod-resize -o jsonpath='{.status.containerStatuses[0].allocatedResources}'
# Output: {"cpu":"200m","memory":"128Mi"}   ← memory request updated
```

---

## Exercise 6 — Describe output

```
Containers:
  app:
    Limits:
      cpu:     400m
      memory:  256Mi
    Requests:
      cpu:     200m
      memory:  128Mi
    Resize Policies:
      cpu:     NotRequired
      memory:  RestartContainer
```

---

## Challenge Answers

**1. CPU NotRequired policy?**
`NotRequired` allows CPU to be resized without restarting the container. The Linux kernel supports adjusting CPU cgroup limits on the fly.

**2. Status Deferred?**
`Deferred` means the node currently doesn't have enough free resources to satisfy the resize request. The kubelet will keep retrying. It does NOT mean the request is rejected permanently — it will be applied when resources become available (e.g., another pod is evicted).

**3. Can you resize init containers in-place?**
No. In-place resize is only supported for regular (app) containers. Init containers cannot be resized in-place.

**4. In-place resize vs rolling update for resource changes?**
- Rolling update: Kubernetes creates new pods with new resource specs, then terminates old pods. This causes pod churn.
- In-place resize: The existing pod is updated directly. No new pod is created, no scheduling delay. This is much faster and cheaper for resource tuning.

**5. More resources than node has?**
The resize status becomes `Infeasible`. The kubelet rejects the resize permanently for that pod on that node. You would need to either reduce the request or move the pod to a larger node.
