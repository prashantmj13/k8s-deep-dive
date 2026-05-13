# 28 — In-place Resize of Pod | Exercises

> **Cluster:** Kubernetes 1.32+ (beta enabled by default) or 1.27+ with feature gate enabled
> **Estimated time:** 20 minutes
> **Namespace:** `resize-lab`

---

## Exercise 1 — Check cluster version and feature support

```bash
kubectl version --short
kubectl get nodes -o wide

# Check if InPlacePodVerticalScaling is available
kubectl explain pod.spec.containers.resizePolicy
```

If the last command returns an error, your cluster does not support in-place resize.

---

## Exercise 2 — Create a pod with resizePolicy

Create `pod-resize.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-resize
  namespace: resize-lab
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        cpu: 100m
        memory: 64Mi
      limits:
        cpu: 200m
        memory: 128Mi
    resizePolicy:
    - resourceName: cpu
      restartPolicy: NotRequired
    - resourceName: memory
      restartPolicy: RestartContainer
```

```bash
kubectl create namespace resize-lab
kubectl config set-context --current --namespace=resize-lab
kubectl apply -f pod-resize.yaml
kubectl get pod pod-resize
```

---

## Exercise 3 — Check current allocated resources

```bash
kubectl get pod pod-resize -o jsonpath='{.spec.containers[0].resources}' | python3 -m json.tool
kubectl get pod pod-resize -o jsonpath='{.status.containerStatuses[0].allocatedResources}' | python3 -m json.tool
```

---

## Exercise 4 — Resize CPU in-place (no restart)

Increase the CPU limit from 200m to 400m:

```bash
kubectl patch pod pod-resize --subresource='resize' \
  --type='merge' \
  -p '{"spec":{"containers":[{"name":"app","resources":{"limits":{"cpu":"400m"},"requests":{"cpu":"200m"}}}]}}'
```

Check status:

```bash
kubectl get pod pod-resize -o jsonpath='{.status.resize}'
kubectl get pod pod-resize -o jsonpath='{.status.containerStatuses[0].allocatedResources}'

# Verify pod did NOT restart (restartCount should still be 0)
kubectl get pod pod-resize
```

---

## Exercise 5 — Resize Memory (triggers container restart)

Increase memory limit:

```bash
kubectl patch pod pod-resize --subresource='resize' \
  --type='merge' \
  -p '{"spec":{"containers":[{"name":"app","resources":{"limits":{"memory":"256Mi"},"requests":{"memory":"128Mi"}}}]}}'
```

Watch the pod:

```bash
kubectl get pod pod-resize -w
```

> **Observe:** RESTARTS count increases by 1 because memory resizePolicy is `RestartContainer`.

---

## Exercise 6 — Describe and verify final state

```bash
kubectl describe pod pod-resize
```

Look for:
- `Resources:` section showing new values
- `resize` field in status

---

## Exercise 7 — Cleanup

```bash
kubectl delete namespace resize-lab
kubectl config set-context --current --namespace=default
```

---

## Challenge Questions

1. Which resize policy allows CPU changes without a container restart?
2. What does status `Deferred` mean during a resize operation?
3. Can you resize an init container in-place?
4. How does in-place resize differ from a Deployment rolling update when changing resources?
5. What happens if you request more resources than the node has available?
